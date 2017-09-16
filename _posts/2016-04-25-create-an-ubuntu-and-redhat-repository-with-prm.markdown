---
layout: post
title: Create an Ubuntu and RedHat repository with PRM
date: '2016-04-25 22:30:46'
---

If you search around the Internet, looking to build your own repository, you'll likely come across something like <a href="https://help.ubuntu.com/community/Repositories/Personal">this</a> or <a href="http://askubuntu.com/questions/170348/how-to-make-my-own-local-repository">this</a>. These tutorials will show you how to build a trivial repository, but if you want to build something that feels and looks more natural, you'll find the Internet sparse on the subject. I ran across a Ruby project though, that fixes this problem. It solves our use case (see below) and it builds our repository tree the way most Ubuntu users are accustomed. It's called <a href="https://github.com/dnbert/prm">PRM</a>.

######Use case
The use case, at least in my case, and I can guess that of many other sysadmins, is that you need to have a repository for a few internal packages. This gives sysadmins a simple way to pull packages on to systems without having to compile them for each individual system.

######INSTALLING PRM DEPENDENCIES

You'll need Ruby 2.0 on your system. You can install it on Ubuntu 14.04 like this:

<pre>sudo apt-get install ruby2.0</pre>


<br>Ruby has it's own package repository system, called <a href="https://rubygems.org/">gems</a>. You can install the PRM gem like this:</br>

<pre>gem2.0 install prm</pre>

<br>You will need a gpg key to sign your packages. Create your GPG key:</br>

<pre>gpg --gen-key</pre>

<br>View your GPG key:</br>

<pre>gpg --list-keys</pre>

<br>If you need more information on creating your key, you can find it <a href="https://www.gnupg.org/gph/en/manual/c14.html">here</a>.</br>
######INSTALLING PRM
Once your gem is installed and your gpg key, move to the directory that you would like to install your repository. Perhaps "/apt" or "/var/www/apt"? And then install your repository:

<pre>prm --type deb --path pool --component prod,dev,staging --release precise,trusty --arch amd64,i386 --gpg &lt;insert your GPG key name here&gt;</pre>

<br>Substitute whichever component suites your needs. I'm using Ubuntu 12.04 (precise) and 14.04 (trusty) for this repository. It's not uncommon to need to install 32 bit applications on 64 bit servers, so I've even added support for i386 in my example. Finally, place the name of your GPG key after the --gpg parameter.</br>
######CONFIGURING HTTP ACCESS
We will naturally want to be able to access our repository. Install Apache.

apt-get install apache2

If you have a DNS alias (CNAME) for your host, create a virtual host entry in a file that has the same alias, for instance "/etc/apache2/sites-available/<strong><em>apt-repo.conf</em></strong>" that looks something along these lines:
<pre>&lt;VirtualHost *:80&gt;
ServerName apt-repo.domain.se
Alias apt-repo
ServerAdmin luke.simmons@domain.se
DocumentRoot /var/www/html/apt-repo
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
&lt;/VirtualHost&gt;</pre>

<br>Create a symbolic link from "/etc/apache2/site-enabled"</br>

<pre>ln -s /etc/apache2/site-available/apt-repo.conf /etc/apache2/site-enabled/</pre>

<br>If you'll be using the same hostname, you can simply update 000-default.conf in "/etc/apache2/site-available". Be sure that <em>DocumentRoot</em> is pointing to the root of your repository! In other words, if you placed your repository under "/apt", <em>DocumentRoot </em>needs to reflect that! You will also need to be sure that index.html is not in your root or you will not be able to list your directories.</br>
<h5>IMPORT OUR PACKAGES</h5>
PRM has build-in support for importing packages, signing them and updating Packages.gz. Place your packages in a directory, let's say /packages and then run the same "prm" command that you did before with the "--directory" parameter. You will need to be in the same catalog that you created your repository.

<blockquote><pre>prm --type deb --path pool --component prod,dev,staging --release precis, trusty --arch amd64 --gpg &lt;insert your GPG key name here&gt; <strong><em>--directory /packages</em></strong></pre></blockquote>

<br>You will need to run this command each time you add a package.
</br>
And voilá. You have a professional grade repository to house  your in-house packages.

Do not forget to import the public GPG key to your servers! And to add your repository name to '/etc/apt/source.d/&lt;my repo name&gt;.list'. Something along the lines of

<pre>deb http://myserver.domain.com trusty dev</pre>

<br>If you want to create an rpm repository for a few Redhat / CentOS machines, PRM can also do that.</br>

<pre>prm --type rpm --path pool --release centos6,centos7 --component stable --arch x86_64 --gpg &lt;insert your GPG key here&gt; --directory /packages</pre>