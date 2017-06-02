---
id: 38
title: Creating an MVC for mod_plsql
date: 2011-03-09T10:47:00+00:00
author: Todd Andrae
layout: post
guid: http://www.toddandrae.com/?p=38
permalink: /creating-a-mvc-for-mod_plsql/
dsq_thread_id:
  - "250608045"
categories:
  - PL/SQL
---
I first got introduced to Oracle&#8217;s mod\_plsql, and PL/SQL in general, a year ago and have spent a good amount of time since then testing the limits of what I can do with it. If you aren&#8217;t familiar with mod\_plsql, it is an Apache module that connects directly to the Oracle database and allows you to run packages that you have stored there and spit out some crass HTML4 pages (this is the default action).<!--more-->

Before I go any further, I need to say this: mod_plsql is not without its faults. You can make it fairly secure, but it is still a connection directly to your database. The user that you use to create your DAD should only have the permissions needed to perform the operations required by your application and should only have access to the packages required to interact with your database! Always filter your data! Never trust user input! Etc!

The first step (assuming you already have Oracle and OHS installed) for working with mod\_plsql is setting up a DAD (database access descriptor). This is a series of directives contained in a location block that are loaded by Apache at runtime. For most installations of Oracle, these directives can be added in the dads.conf file located in your ORACLE\_HOME/apache/modplsql/ directory.

<pre class="brush: plain; title: ; notranslate" title="">SetHandler pls_handler
Order deny,allow
Allow from all
AllowOverride All
PlsqlDatabaseUsername mvc_user
PlsqlDatabasePassword mvc_password
PlsqlDatabaseConnectString 127.0.0.1:1521:orcl
PlsqlAuthenticationMode PerPackageOwa
</pre>

There are only three lines that you will need to change from this: PlsqlDatabaseUsername, PlsqlDatabasePassword, PlsqlDatabaseConnectString.  The database connect string can be the short name (orcl) or it can be the full string as seen above.  After putting this information in, run the dadTool.pl script located in the same directory. This will obfuscate the password in the conf file and gives you a feeling of security. Use opmnctl to restart the OHS service.  If you attempt to load the URL (http://127.0.0.1/plsql_mvc or however it maps on your system), you should get a 404. This is a good sign.

So let&#8217;s start with the package spec for our plsql_mvc:

<pre class="brush: sql; title: ; notranslate" title="">create or replace package plsql_mvc as
function authorize return boolean;
procedure controller(name_array in owa.vc_arr, value_array in owa.vc_arr);
end plsql_mvc;</pre>

Looks pretty simple.  Four lines, one function and one procedure. I&#8217;m going to go back to basics for those that started out like me (no knowledge of mod_plsql at all). In our dads.conf, we set a directive for PlsqlAuthenticationMode to be PerPackageOwa. What this enables is a function (always called authorize) that is executed everytime a procedure is called from our package. If this function returns a boolean false, then the http response of 401 unauthorized is returned before further processing is done.

Another nice feature that took me a while to track down is the use of name\_array and value\_array. These two variables pass in an array of the variable names and variable values. It doesn&#8217;t make total sense to pass in two arrays to handle the passed variables, but you work with what you are given. The caveat with these passed parameters is that they are only available if you enable flexible parameter passing. This is a fancy way of saying, you prepend the package name in your URL with an exclamation point (!). This comes in handy since Oracle procedures and functions expect to know everything being passed in or else they fail with a signature parameter mismatch error.

<pre class="brush: sql; title: ; notranslate" title="">create or replace package body plsql_mvc as
function authorize return boolean is
begin
htp.print('authorize');
return true;
end authorize;

procedure controller(name_array in owa.vc_arr, value_array in owa.vc_arr) is
begin
htp.print('controller');
end controller;

begin
htp.print('block');
end plsql_mvc;</pre>

The package body looks just as simple as the specification. The only change in flow is in a begin statement after the controller procedure. This block of code is the initialization block and is executed whenever a procedure or function is called from this package. This adds yet another very important layer to our prototype.

Now, earlier, I said that the PerPackageOwa directive automatically runs the authorize function before any other piece of code. That was a lie. In actuality, the initialization part gets run first and foremost before any security or routing has been performed. To see how this is processed direct your browser to your configured DAD, i.e. http://127.0.0.1/plsql\_mvc/!plsql\_mvc.controller. You should get something that looks like this:

<pre class="brush: plain; title: ; notranslate" title="">block
authorize
controller</pre>

At this point, our MVC is nothing more than a 10k package that takes input and does nothing. Its a long way from finished, but the shell is almost there. Next let&#8217;s add a new data type for our associative array, a private function to swap the variables into the array and retrieve them from the array.

In our package spec add the following before authorize function:

<pre class="brush: sql; title: ; notranslate" title="">type associative_array is table of varchar2(256) indexed by varchar2(256);</pre>

With that done, we now need a way to populate that array. In our package body, before our authorize function, we are going to add a set of private functions and a private variable.

<pre class="brush: sql; title: ; notranslate" title=""> passed_variables associative_array;
function populate_array(name_array in owa.vc_arr, value_array in owa.vc_arr) return boolean;
function set_value_of(key in varchar2, value in varchar2) return boolean;
function get_value_of(key in varchar2) return varchar2;</pre>

Declaring your private functions and procedures first before using them is another trick that I wish was told to everyone the first day they started working with PL/SQL. Doing this allows you to keep your initial package spec cleaner and allows you to reference private functions internally regardless of the order you declare them in.

Now that the declaration is out of the way, we need to define the actual code in the private functions.

<pre class="brush: sql; title: ; notranslate" title=""> function populate_array(name_array in owa.vc_arr, value_array in owa.vc_arr) return boolean is
begin
for i in 1.. name_array.count loop
if set_value_of(name_array(i), value_array(i)) then
null;
   else
     return false;
end if;
end loop;
return true;
end;

function set_value_of(key in varchar2, value in varchar2) return boolean is
begin
passed_variables(key) := value;
if passed_variables(key) = value then
return true;
else
return false;
end if;
end;

function get_value_of(key in varchar2) return varchar2 is
begin
if passed_variables.exists(key) then
return passed_variables(key);
else
return null;
end if;
end;</pre>

Our first function, populate\_array, does exactly what you think it might do: populates an array. With the name\_array and value\_array being passed in, the variable name is matched with the same index to the value\_array. We loop through the number of values in name\_array and then call a function, set\_value\_of, to do the actual setting. I decided to split to split out this function since I thought it could be a useful private function later down the road. The set\_value_of function takes the passed in key, creates an index in our passed variables table with this key and then sets the value passed in. Next, it just does a simple spot check to make sure that the passed value is equal to the value in table. It hasn&#8217;t failed with a false yet, but I&#8217;m sure it will some day.

The last function is the get\_value\_of function. This function takes a key passed in and returns the value in the table. If the key is not found, it returns a null. There will probably be some question as to why I chose to store and return varchar2s instead of some other datatype. When information is passed from HTML forms or via HTTP, there isn&#8217;t really any designation as to the type of information being passed, which means it is all plain text. When we actually go to use the variable, we will cast it to the type that we need.

We now have three private functions, but they don&#8217;t really do anything yet. In our controller procedure, add a boolean variable in the declaration called passed\_variable\_inited and set the default value to the return value of our populate_array function

<pre class="brush: sql; title: ; notranslate" title="">passed_variables_inited boolean := populate_array(name_array, value_array);</pre>

Next enclose our htp.print(&#8216;controller&#8217;) section in the controller with an if statement

<pre class="brush: sql; title: ; notranslate" title="">  if passed_variables_inited then
htp.print('controller');
end if;</pre>

If for some reason our variables that have been passed in cannot be inserted into our passed_variables table, our function will return a false and no further processing will be done.

And that is the very basic building block for our framework. In the next installment, we will look at the flow of router -> security -> controller and modify our passed variables to differentiate between POST and GET variables.