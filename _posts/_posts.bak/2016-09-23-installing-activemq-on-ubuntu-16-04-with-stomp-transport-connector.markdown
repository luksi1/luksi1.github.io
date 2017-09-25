---
layout: post
title: Installing ActiveMQ on Ubuntu 16.04 for use with Mcollective
date: '2016-09-23 14:23:40'
categories: [Puppet]
---

I was getting a lot of errors while attempting to install an ActiveMQ server for use with Mcollective. Following the Puppet docs for Mcollective [here](https://docs.puppet.com/mcollective/deploy/standard.html), I got several errors and eventually had to abandon Ubuntu's ActiveMQ repository package. This is due to the fact that a mess of features are missing. 

From the README.Debian:

```
Disabled features
-----------------
This package doesn't contains (yet) all features provided by upstream
as some dependencies are missing from Debian.
For a list of disabled modules, you can look at
  /usr/share/doc/libactivemq-java/README.Debian
```

Simply install the tarball from ActiveMQ under /opt. I was able to get everything setup with the following module, taken from the MCollective module's examples.

```
# This class prepares an ActiveMQ middleware service for use by MCollective.
#
# The default parameters come from the mco_profile::params class for only one
# reason. It allows the user to OPTIONALLY use Hiera to set values in one place
# and have them propagate multiple related classes. This will only work if the
# parameters are set in Hiera. It will not work if the parameters are set from
# an ENC.
#
class profile::app::middleware::activemq (
  $memoryusage               = '200 mb',
  $storeusage                = '1 gb',
  $tempusage                 = '1 gb',
  $console                   = true,
  $ssl_ca_cert               = 'puppet:///modules/profile/mcollective/certs/ca.pem',
  $ssl_server_cert           = 'puppet:///modules/profile/mcollective/certs/<SIGNED CERT>.pem',
  $ssl_server_private        = 'puppet:///modules/profile/mcollective/private_keys/<PRIVATE KEY>.pem',
  $middleware_user           = 'mcollective',
  $middleware_password       = 'secret',
  $middleware_admin_user     = 'admin',
  $middleware_admin_password = 'secret',
  $middleware_ssl_port       = '61614'
) {

  $confdir = '/opt/apache-activemq-5.14.0/conf'

  file { "${confdir}/activemq.xml":
	owner => 'activemq',
	group => 'activemq',
	mode => '0444',
        content => template('profile/activemq/activemq_template.erb'),
   }

  # Set up SSL configuration. Use copies of the PEM keys specified to create
  # the Java keystores.
  file { "${confdir}/ca.pem":
    owner   => 'activemq',
    group   => 'activemq',
    mode    => '0444',
    source  => $ssl_ca_cert,
  }
  file { "${confdir}/server_cert.pem":
    owner   => 'activemq',
    group   => 'activemq',
    mode    => '0444',
    source  => $ssl_server_cert,
  }
  file { "${confdir}/server_private.pem":
    owner   => 'activemq',
    group   => 'activemq',
    mode    => '0400',
    source  => $ssl_server_private,
  }

  java_ks { 'mcollective:truststore':
    ensure       => 'latest',
    certificate  => "${confdir}/ca.pem",
    target       => "${confdir}/truststore.jks",
    password     => 'puppet',
    trustcacerts => true,
    require      => File["${confdir}/ca.pem"],
  } ->

  file { "${confdir}/truststore.jks":
    owner   => 'activemq',
    group   => 'activemq',
    mode    => '0400',
    before  => Java_ks['mcollective:keystore'],
  }

  java_ks { 'mcollective:keystore':
    ensure       => 'latest',
    certificate  => "${confdir}/server_cert.pem",
    private_key  => "${confdir}/server_private.pem",
    target       => "${confdir}/keystore.jks",
    password     => 'puppet',
    trustcacerts => true,
    require      => [
      File["${confdir}/server_cert.pem"],
      File["${confdir}/server_private.pem"],
    ],
  } ->
  file { "${confdir}/keystore.jks":
    owner   => 'activemq',
    group   => 'activemq',
    mode    => '0400',
  }

}
```

And the template...

```
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:amq="http://activemq.apache.org/schema/core"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd
  http://activemq.apache.org/camel/schema/spring http://activemq.apache.org/camel/schema/spring/camel-spring.xsd">

    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
<!--        <property name="locations">
            <value>file:${activemq.base}/conf/credentials.properties</value>
        </property> -->
    </bean>

    <!--
      For more information about what MCollective requires in this file,
      see http://docs.puppetlabs.com/mcollective/deploy/middleware/activemq.html
    -->

    <!--
      WARNING: The elements that are direct children of <broker> MUST BE IN
      ALPHABETICAL ORDER. This is fixed in ActiveMQ 5.6.0, but affects
      previous versions back to 5.4.
      https://issues.apache.org/jira/browse/AMQ-3570
    -->
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" useJmx="true" schedulePeriodForDestinationPurge="60000" persistent="false">
        <!--
          MCollective generally expects producer flow control to be turned off.
          It will also generate a limitless number of single-use reply queues,
          which should be garbage-collected after about five minutes to conserve
          memory.

          For more information, see:
          http://activemq.apache.org/producer-flow-control.html
        -->
        <destinationPolicy>
          <policyMap>
            <policyEntries>
              <policyEntry topic=">" producerFlowControl="false"/>
              <policyEntry queue="*.reply.>" gcInactiveDestinations="true" inactiveTimoutBeforeGC="300000" />
            </policyEntries>
          </policyMap>
        </destinationPolicy>

        <managementContext>
            <managementContext createConnector="false"/>
        </managementContext>

        <plugins>
          <statisticsBrokerPlugin/>

          <!--
            This configures the users and groups used by this broker. Groups
            are referenced below, in the write/read/admin attributes
            of each authorizationEntry element.
          -->
          <simpleAuthenticationPlugin>
            <users>
              <authenticationUser username="<%= @middleware_user %>" password="<%= @middleware_password %>" groups="mcollective,everyone"/>
              <authenticationUser username="<%= @middleware_admin_user %>" password="<%= @middleware_admin_password %>" groups="mcollective,admins,everyone"/>
            </users>
          </simpleAuthenticationPlugin>

          <!--
            Configure which users are allowed to read and write where. Permissions
            are organized by group; groups are configured above, in the
            authentication plugin.

            With the rules below, both servers and admin users belong to group
            mcollective, which can both issue and respond to commands. For an
            example that splits permissions and doesn't allow servers to issue
            commands, see:
            http://docs.puppetlabs.com/mcollective/deploy/middleware/activemq.html#detailed-restrictions
          -->
          <authorizationPlugin>
            <map>
              <authorizationMap>
                <authorizationEntries>
                  <authorizationEntry queue=">" write="admins" read="admins" admin="admins" />
                  <authorizationEntry topic=">" write="admins" read="admins" admin="admins" />
                  <authorizationEntry topic="mcollective.>" write="mcollective" read="mcollective" admin="mcollective" />
                  <authorizationEntry queue="mcollective.>" write="mcollective" read="mcollective" admin="mcollective" />
                  <!--
                    The advisory topics are part of ActiveMQ, and all users need access to them.
                    The "everyone" group is not special; you need to ensure every user is a member.
                  -->
                  <authorizationEntry topic="ActiveMQ.Advisory.>" read="everyone" write="everyone" admin="everyone"/>
                </authorizationEntries>
              </authorizationMap>
            </map>
          </authorizationPlugin>
        </plugins>


        <sslContext>
            <sslContext
                keyStore="keystore.jks" keyStorePassword="puppet"
                trustStore="truststore.jks" trustStorePassword="puppet"
            />
        </sslContext>

        <!--
          The systemUsage controls the maximum amount of space the broker will
          use for messages. For more information, see:
          http://docs.puppetlabs.com/mcollective/deploy/middleware/activemq.html#memory-and-temp-usage-for-messages-systemusage
        -->
        <systemUsage>
            <systemUsage>
                <memoryUsage>
                    <memoryUsage limit="<%= @memoryusage %>"/>
                </memoryUsage>
                <storeUsage>
                    <storeUsage limit="<%= @storeusage %>" name="foo"/>
                </storeUsage>
                <tempUsage>
                    <tempUsage limit="<%= @tempusage %>"/>
                </tempUsage>
            </systemUsage>
        </systemUsage>

        <!--
          The transport connectors allow ActiveMQ to listen for connections over
          a given protocol. MCollective uses Stomp, and other ActiveMQ brokers
          use OpenWire. You'll need different URLs depending on whether you are
          using TLS. For more information, see:

          http://docs.puppetlabs.com/mcollective/deploy/middleware/activemq.html#transport-connectors
        -->
        <transportConnectors>
            <transportConnector name="stomp+nio+ssl" uri="stomp+ssl://0.0.0.0:<%= @middleware_ssl_port %>?needClientAuth=true"/>
	<!--
	    <transportConnector name="stomp+nio+ssl" uri="stomp+nio+ssl://0.0.0.0:<%= @middleware_ssl_port %>?needClientAuth=false&amp;transport.enabledProtocols=TLSv1,TLSv1.1,TLSv1.2"/>
	-->
        </transportConnectors>
    </broker>

    <% if @console %>
    <!--
      Enable web consoles, REST and Ajax APIs and demos.
      It also includes Camel (with its web console); see ${ACTIVEMQ_HOME}/conf/camel.xml for more info.

      See ${ACTIVEMQ_HOME}/conf/jetty.xml for more details.
    -->
    <import resource="jetty.xml"/>
    <% end %>
</beans>
```

The MCollective server required a bit of adjustments, but more or less, worked out of the box. Notice that I needed to enable the module in the /etc/default/mcollective file.

I also had my client certificates on my puppet master under /etc/puppet/agent/ssl insteaad of /var/lib/puppet/ssl. If you run into problems with an MCollective client, you might want double-check your `ssldir` with `puppet config print ssldir`.



```
  $ssldir = '/var/lib/puppet/ssl'

  file { '/etc/default/mcollective':
    owner   => 'root',
    group   => 'root',
    mode    => '0644',
    ensure  => 'present',
    content => 'RUN=yes',
  } ->
  class { '::mcollective':
    middleware_hosts          => ['puppet.vgregion.se'],
    middleware_port           => '61613',
    middleware_ssl_port       => '61614',
    middleware_ssl            => true,
    middleware_ssl_cert       => "${ssldir}/certs/${::clientcert}.pem",
    middleware_ssl_key        => "${ssldir}/private_keys/${::clientcert}.pem",
    middleware_ssl_ca         => "${ssldir}/certs/ca.pem",
    middleware_user           => 'mcollective',
    middleware_password       => 'secret',
    middleware_admin_user     => 'admin',
    middleware_admin_password => 'secret',
    securityprovider          => 'ssl',
    ssl_client_certs          => 'puppet:///modules/profile/mcollective/client_certs',
    ssl_ca_cert               => 'puppet:///modules/profile/mcollective/certs/ca.pem',
    ssl_server_public         => 'puppet:///modules/profile/mcollective/certs/mcollective-servers.pem',
    ssl_server_private        => 'puppet:///modules/profile/mcollective/private_keys/mcollective-servers.pem',
  }
```
