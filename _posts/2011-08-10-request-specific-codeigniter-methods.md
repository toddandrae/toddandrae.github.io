---
id: 95
title: Request Specific CodeIgniter Methods
date: 2011-08-10T08:14:21+00:00
author: Todd Andrae
layout: post
guid: http://www.toddandrae.com/?p=95
permalink: /request-specific-codeigniter-methods/
dsq_thread_id:
  - "382219893"
ratings_users:
  - "0"
ratings_score:
  - "0"
ratings_average:
  - "0"
categories:
  - CodeIgniter
  - PHP
---
When I was looking through frameworks to use, I was weighing some of the choice based on the frameworks ability to route calls based on HTTP Request Methods. Call me a pedant, but I believe that a POST to a page should be a whole different plate of spaghetti from a GET of the page. GET&#8217;d pages should get and retrieve information and POST&#8217;d pages should perform actions.<!--more-->

From the MVCs available, there weren&#8217;t any available (that I saw), that off the shelf allowed for this kind of functionality. I chose CodeIgniter and figured I would just have separate named methods for POST or GET.

As I said in a previous post, my exposure to CodeIgniter was based on old tutorials I had pulled from various tutorial sites. Once, I found some more recent tutorials that led to many an aha moment. This is one of those moments.

By extending the CI\_Router class with our MY\_Router class, we can modify the fetch\_method method to return our method name and the HTTP Request Method. In 9 lines of code, you can now have an index\_get method and an index_post method to handle exactly what they need to handle.

Add  this to or create your /application/core/MY_Controller.php file

<pre class="brush: php; title: ; notranslate" title="">class MY_Router extends CI_Router {
  function fetch_method(){
    $request = strtolower($_SERVER['REQUEST_METHOD']);
    if ($this-&gt;method == $this-&gt;fetch_class()) {
      $method = 'index_' . $request;
    } else {
      $method = $this-&gt;method . '_' . $request;
    }
    return $method;
  }
}
</pre>

Next, just identify the Request Method you would like to allow on your class methods by adding &#8220;\_get&#8221; or &#8220;\_post&#8221; to the end of the method name. That should be it. You can now create individual methods that look like this:

<pre class="brush: php; title: ; notranslate" title="">public function index_post() {
  /*Drop database schema*/
  redirect(uri_string(), 'location', 301);
}
public function index_get() {
  /*Display what just happened*/
}
</pre>

This prevents the POST&#8217;d page from being refreshed and keeps the retrieval logic separate. I haven&#8217;t tested how CI handles HEAD requests, but the next step is to keep logic from being performed and only 404 if the database cannot be connected.