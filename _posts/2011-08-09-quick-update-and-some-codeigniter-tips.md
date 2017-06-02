---
id: 85
title: Quick Update and Some CodeIgniter Tips
date: 2011-08-09T09:11:48+00:00
author: Todd Andrae
layout: post
guid: http://www.toddandrae.com/?p=85
permalink: /quick-update-and-some-codeigniter-tips/
dsq_thread_id:
  - "381298558"
categories:
  - CodeIgniter
  - PHP
---
So a lot has changed in the past few months. I won&#8217;t bore you with details, but I did move back over to the east coast and no longer work with an Oracle based company. I started back with a previous employer, a multi-office real estate company, and was tasked with getting everything back up to speed. This meant examining everything that they were previously doing and streamlining the whole shebang. In doing that, I decided to scrap the wordpress/drupal/open-realty mashup that had been put in place and build a ground up system that could seamlessly share information between all the different areas.<!--more-->

The first few days, I was filled with programmer&#8217;s pride and was dead set on designing my own custom MVC framework. Soon, I came to my senses and realized that I had to put pride aside and put something in place that had a proven track record and would yield more time to concentrate on the business logic and features required. I had become familiar with CodeIgniter through some interviews in the Denver area and decided to load it up. It was an immediate friendship.

The folks that contribute to CodeIgniter have thought this whole thing out pretty well. I came into it and started tearing apart the system folder, mucking up the core libraries and generally just causing total code anarchy based on a few tutorials I had seen online. Don&#8217;t do this. Most of the tutorials are old and there seems to have been a huge jump in recent versions to move the application folder out of the system folder. This makes thing so much better.

Now, let&#8217;s get down to business! The first objective I had to tackle with CodeIgniter was how to share my models, and occasionally views, across multiple subdomains and applications. I finally stumbled on the answer by using a combination of Apache virtual hosts, custom php landing pages and an extended router class.

First, the Apache changes:

<pre class="brush: plain; title: ; notranslate" title="">&lt;VirtualHost *:80&gt;
  DocumentRoot "/var/www/"
  ServerName localhost
  ServerAlias blog.domain.tld
  &lt;Directory "/var/www/"&gt;
    Options +Indexes
    Options FollowSymLinks
    AllowOverride All
    &lt;IfModule mod_rewrite.c&gt;
      RewriteEngine On
      RewriteBase /
      RewriteCond %{REQUEST_URI} ^system.*
      RewriteRule ^(.*)$ blog.php/$1 [L]
      RewriteCond %{REQUEST_URI} ^application.*
      RewriteRule ^(.*)$ blog.php/$1 [L]
      RewriteCond %{REQUEST_FILENAME} !-f
      RewriteCond %{REQUEST_FILENAME} !-d
      RewriteRule ^(.*)$ blog.php?/$1 [L]
    &lt;/IfModule&gt;
    &lt;IfModule !mod_rewrite.c&gt;
      ErrorDocument 404 /index.php
    &lt;/IfModule&gt;
  &lt;/Directory&gt;
&lt;/VirtualHost&gt;
</pre>

Currently, each subdomain has to be set up with custom rewrite rules. I am going to come back to this later and see if I can&#8217;t do it with a *.domain.tld virtualhost.

Next the little php snippet:

<pre class="brush: php; title: ; notranslate" title="">&lt;?php define('SUBDOMAIN', 'blog'); require_once 'index.php' ?&gt;
</pre>

That&#8217;s all there is to the php file. We are just going to define a constant for our subdomain for use later.

And then our extended router class:

<pre class="brush: php; title: ; notranslate" title="">&lt;?php  if ( ! defined('BASEPATH')) exit('No direct script access allowed');
  class MY_Router extends CI_Router {
    function _set_request($segments = array()){
      if(SUBDOMAIN === 'blog') {
        $controller_directory = 'blog';
      } else {
        $controller_directory = '';
      }
      $segments = array_reverse($segments);
      $segments[] = $controller_directory;
      $segments = array_reverse($segments);
      parent::_set_request($segments);
    }
  }
</pre>

And this is the meat. Place this code in a file called MY\_Router.php in your /application/core/ directory. This extends the base router and allows us to add our &#8216;controller directory&#8217; to the segment array and pass that back to the  CI\_Router \_set\_request method to trick it into thinking it is already in the URI. With that new segment in there, CodeIgniter goes to our application folder, does a check for is_dir and grabs our controller files from the subdirectory we inserted. Nothing to it! And this way, I get to share my models amongst my controllers without having to rewrite them in separate application folders or using symbolic links.

Granted, this might not amaze some of the current users of CodeIgniter, but this immediately sold me on CI&#8217;s usage as my MVC of choice and led me to the next step of setting POST and GET specific methods as well as custom subdomains for users, but I&#8217;ll save those for next time.