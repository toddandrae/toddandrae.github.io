---
id: 262
title: Preventing 502/520s with CodeIgniter and MaxCDN/CloudFlare
date: 2014-12-20T15:09:52+00:00
author: Todd Andrae
layout: post
guid: http://www.toddandrae.com/?p=262
permalink: /preventing-502520s-with-codeigniter-and-maxcdncloudflare/
ratings_users:
  - "0"
ratings_score:
  - "0"
ratings_average:
  - "0"
dsq_thread_id:
  - "3344319535"
categories:
  - Uncategorized
---
Where I work, we finally outgrew our shared hosting britches and moved over to a VPS. Most of this growth came not from the need for processing power, but from our need to store a large number of images. The standard real estate listing went from averaging 8 images to 15 and storing multiple file sizes led to us needing to store 120,000 images. The plan was to activate [ludicrous speed](http://code.tutsplus.com/tutorials/activating-ludicrous-speed-combine-cloudflare-with-a-cdn-on-your-blog--wp-24586).

<!--more-->

I got everything set up, RackSpace for hosting, used GlobalSign for SSL, CloudFlare for DNS and MaxCDN for images. Everything went well and we immediately saw a decrease in first byte time and overall page load times were decreased. I noticed some intermittent issues with both CloudFlare and MaxCDN, but the nginx access logs showed that everything was being resolved fine and ending with a 200 status code. I moved the main application back to the shared host but left resolving images with MaxCDN and started troubleshooting.

MaxCDN support thought the issue could be with our certificate so I set down the path of testing the SSL configuration. I used [SSLLabs](https://www.ssllabs.com/ssltest/) and a [tutorial from Remy van Elst](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html) to get a solid A+ rating. I ran a battery of other SSL tests against the server and they all agreed that SSL was solid. I went back to MaxCDN and they ran a few more tests and insisted I add the Root CA to the certificate chain. I went back to GlobalSign Support to get their take. They didn&#8217;t think adding the Root CA to the chain would make a difference, but they sent the cert and I added it. My SSL rating went to an A for containing an anchor in the chain, but if it solves MaxCDN 502 issue, I&#8217;ll take it. I purged the cache and ran it again. 502!

After some troubleshooting, I found that I could 502 in Chrome, but in Firefox I would get the file. But I would only get the first file. Additional files in Firefox would 502. I jumped over to the command line to cURL the image URLs. Each and every time the file would be resolved and returned. I jumped back to Chrome and opened an incognito tab. 200! I attempted a second file and returned a 502. I compared the response headers across the various attempts as well as accessing the files directly. That is when it hit me. Cookies!

I went back to MaxCDN with my theory of the number and size of cookies being sent by CodeIgniter causing the 502. I was told it had to be because the server wasn&#8217;t performing TLS 1.1/1.2 properly and that our host was blocking their requests and that they were going to continue looking in to the issue. At this point, I gave up on MaxCDN providing a solution, so I went about testing my own solution. I searched for other persons having the same issue and came across a [thread](https://ellislab.com/forums/viewthread/228023/) on EllisLabs that had the [exact solution I needed](https://ellislab.com/forums/viewthread/228023/#1035436). I added the custom session class and the post_controller hook and the 502s disappeared. Completely. I turned CloudFlare back on and the 520s disappeared as well.

So, if you are having a 502 error with MaxCDN or a 520 with CloudFlare and can&#8217;t figure out where it is coming from, check your cookies. Despite the blue muppet&#8217;s persistence, there is a chance that you can have too many cookies or your cookies may be too big.

**UPDATE:** The Ellis Lab forum links are dead, so here is the solution to the problem. I haven&#8217;t ran CI 3.0 yet, so I don&#8217;t know if this update is required there or not.

In your application/libraries directory, extend CI_Session. For my install, my prefix is RS.

<pre class="brush: php; title: ; notranslate" title="">class RS_Session extends CI_Session {
	private $cookie_data = array();
	public function _set_cookie($cookie_data = NULL){
		if(is_null($cookie_data)){
			$cookie_data = $this-&gt;userdata;
		}
		$cookie_data = $this-&gt;_serialize($cookie_data);
		if($this-&gt;sess_encrypt_cookie == true){
			$cookie_data = $this-&gt;CI-&gt;encrypt-&gt;encode($cookie_data);
		} else {
			$cookie_data = $cookie_data . md5($cookie_data . $this-&gt;encryption_key);
		}
		$this-&gt;cookie_data[] = $cookie_data;
	}
	public function finalize_session(){
		if(!empty($this-&gt;cookie_data)){
			$cookie_data = array_pop($this-&gt;cookie_data);
			$expire = ($this-&gt;sess_expire_on_close === true) ? 0 : $this-&gt;sess_expiration + time();
			setcookie(
				$this-&gt;sess_cookie_name,
				$cookie_data,
				$expire,
				$this-&gt;cookie_path,
				$this-&gt;cookie_domain,
				$this-&gt;cookie_secure
			);
			$this-&gt;cookie_data = array();
		}
	}</pre>

You also need to add a hook to finalize the session. I created a file, &#8220;finalizeSession.php&#8221;, in my application/hooks directory. 

<pre class="brush: php; title: ; notranslate" title="">function finalize_session(){
	$CI =& get_instance();
	$CI-&gt;session-&gt;finalize_session();
}</pre>

And then added this to the application/config/hooks.php file

<pre class="brush: php; title: ; notranslate" title="">$hook['post_controller'][] = array(
    "class" =&gt; "",
    "function" =&gt; "finalize_session",
    "filename" =&gt; "finalizeSession.php",
    "filepath" =&gt; "hooks"
);</pre>