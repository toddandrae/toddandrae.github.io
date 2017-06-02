---
id: 111
title: Wildcard Domains, Nginx and DNSMasq on OS X 10.10 Yosemite
date: 2014-10-19T14:45:05+00:00
author: Todd Andrae
layout: post
guid: http://www.toddandrae.com/?p=111
permalink: /wildcard-domains-nginx-and-dnsmasq-on-os-x-10-10-yosemite/
ratings_users:
  - "0"
ratings_score:
  - "0"
ratings_average:
  - "0"
dsq_thread_id:
  - "3134427734"
categories:
  - Mac OS X
  - nginx
---
So I&#8217;m sure everyone that is showing up here recently blew away their entire system to perform a clean install of 10.10 and now you need to get your devving back on. This is the first part of my dev set up. In the next session, I&#8217;ll go over signing certs and setting up a VPN locally for mobile testing.

<!--more-->

One thing I always forget until I need it is to show hidden files. You may want to do that before you get started. Or you can wait until you forget and need it later like me. Your choice.

<pre>defaults write com.apple.finder AppleShowAllFiles -boolean true ; killall Finder</pre>

## Install XCode and Command Line Developer Tools

According to the [Homebrew wiki](https://github.com/Homebrew/homebrew/wiki/Installation), you don&#8217;t need the XCode IDE, but I was already downloading it for native app development and decided to include it. Feel free to skip this step and post your results if things go awry. You will still need to grab the Command Line Tools (or you can let Homebrew grab those for you).

For the moment the XCode in the App Store is version 6.01. To get 6.1, download it from <https://developer.apple.com/downloads/index.action>. It will hopefully live at <https://itunes.apple.com/us/app/xcode/id497799835> in the coming days.

You will need to download the XCode Command Line Developer Tools. You can grab the 6.1 version from <https://developer.apple.com/downloads/index.action?name=for%20Xcode%20-#>.

## Install <a href="http://brew.sh/" target="_blank">Homebrew</a>

Now it is time to launch Terminal and start copying and pasting. First we need to grab Homebrew. Paste the following in to your terminal window.

<pre>ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"</pre>

That should have completed with no warnings or messages. Once finished run

<pre>brew doctor</pre>

Again, that should have run with 0 warnings. If you are warning-less at this point, congratulations, you can move on to installing your brews.

## Brew DNSMasq

DNSMasq, in conjunction with resolver will allow all hostnames with a .dev TLD to resolve to your local machine. If you are looking for an in-depth description of what is going on, check out [Thomas Sutton&#8217;s post on installing DNSMasq using Homebrew](http://passingcuriosity.com/2013/dnsmasq-dev-osx/) (I actually borrowed his tee command for this post)

<pre>brew install dnsmasq</pre>

I don&#8217;t remember having to do this with Mavericks or lower, but with Yosemite, I had to first create the /usr/local/etc directory before creating my configuration files. Even if you don&#8217;t use DNSMasq, you may still need to create the directory for other formulae.

<pre>mkdir /usr/local/etc</pre>

Go ahead and copy the dnsmasq example configuration file to our new directory

<pre>cp /usr/local/opt/dnsmasq/dnsmasq.conf.example /usr/local/etc/dnsmasq.conf</pre>

Edit your file and add this line. This will direct all lookups for hostnames with a top level domain of dev to resolve to 127.0.0.1. If you want to use a different top level domain name, just replace dev with your chosen TLD.

<pre>address=/dev/127.0.0.1</pre>

Next make a link for the provided plists.

<pre>ln -sfv /usr/local/opt/dnsmasq/*.plist /Library/LaunchDaemons</pre>

Next we need to tell OS X where to resolve our .dev addresses. This is unbelievably easy. First create a directory for resolver in /etc/

<pre class="sourceCode bash"><code class="sourceCode bash">&lt;span class="kw">sudo&lt;/span> mkdir -p /etc/resolver</code></pre>

Then create a file named dev. If you went with another test level domain, use that for your filename instead.

<pre class="sourceCode bash"><code class="sourceCode bash">&lt;span class="kw">sudo&lt;/span> tee /etc/resolver/dev &lt;span class="kw">&gt;&lt;/span>/dev/null &lt;&lt;EOF
nameserver 127.0.0.1
EOF</code></pre>

Launch dnsmasq

<pre>sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist</pre>

## [Brew Nginx-Full](https://github.com/Homebrew/homebrew-nginx)

I started using this branch of nginx on my previous box because I needed to build with http-additions.

<pre>brew tap homebrew/nginx</pre>

The advantage of this formula is the available options. Run the following command to find out what you have available.

<pre>brew options nginx-full</pre>

To install, run the following. (Omit the square brackets, if you just need a base install)

<pre>brew install nginx-full [--with-additions]</pre>

Next, you probably want to run on port 80 so we need to do some ownership and permissions changes.

<pre>sudo chown root:wheel /usr/local/Cellar/nginx-full/1.6.2/bin/nginx
sudo chmod u+s /usr/local/Cellar/nginx-full/1.6.2/bin/nginx</pre>

Create links for the provided plists

<pre>ln -sfv /usr/local/opt/nginx-full/*.plist ~/Library/LaunchAgents</pre>

To start on port 80 edit nginx.conf located in /usr/local/etc/nginx/ and change the server section to resemble the following

<pre>listen 80;
server_name *.dev;</pre>

You are ready to start nginx

<pre>sudo nginx</pre>

## [Brew PHP-FPM](https://github.com/Homebrew/homebrew-php) {.p1}

Brewing PHP-FPM is fairly straightforward. Run these two commands and you should be off and running.

<pre>brew install php56 --with-fpm --with-homebrew-curl --with-homebrew-openssl --withimap --withpgsql</pre>

<pre>ln -sfv /usr/local/Cellar/php56/5.6.1/homebrew.mxcl.php56.plist ~/Library/LaunchAgents/</pre>