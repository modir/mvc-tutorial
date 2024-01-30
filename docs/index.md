# Building a simple MVC system with PHP7 and higher

## Abstract

This tutorial was written based on another (simpler) tutorial
written by Dennis Pallett in PHPit.com.

The framework which we have at the end is fully working but we advice
against using it in production as we miss a lot of security features.

## Introduction

In this tutorial you will learn how to build a simple Model-View-
Controller system with PHP 7 and some of SPL's (Standard PHP
Library) features. I'll take you through all the steps necessary
to start from scratch to a full-blown MVC system. We will look not only
at MVC but at other common design patterns as well.

## Folders

In our little system we will create the following folders, we will
create them when needed:

- `classes`
Here will be our base classes plus the framework.

- `controllers`
Here will be our controllers which handle the data between the
models and the views.

- `models`
In this we will place our models that are the basis for our data.
The model defines the data and how it is saved.

- `views`
In here we have our view files, we use them to present the data
and information to the user. The view (or multiple views) is
what you see when you visit a website.

## One point of entry

One of the important things about our MVC system is that it will
have only a single point of entry. Instead of having several dozens of
PHP files that do the following:

~~~php
<?php
include('global.php');
// Rest of the actual page code here
~~~

We will have a single page that handles all the requests. This means
we won't have to include a global.php every time we want to create
a new page. This "single point of entry" page will be called
'index.php' and, at the moment, looks like this:

~~~php
<?php
// Do something
~~~

As you can see, the index page does nothing yet, but it will in a
minute.

To make sure that all the requests go to the index page we will
setup an .htaccess RewriteRule using the mod_rewrite engine. Put
the following code in a file called ".htaccess" in the same directory
as the index.php file:

~~~
RewriteEngine on
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php?route=$1 [L,QSA]
~~~

First we check if the actual file exists using the RewriteCond
command, and if it doesn't exist, we redirect it through the
index.php file. We have to check if the file exists, because we also
want to be able to use regular non-PHP files, such as JPEG images.
If you can't use .htaccess or mod_rewrite, you will have to manually
redirect the requests through the index.php file, which means all
your links must be in the form of "index.php?route=[request-goes-
here]", e.g. index.php?route=chat/index.

Now that all the requests are going through a single point of entry,
we can start writing the index.php file. The first thing we will have
to do is some startup tasks. Create a new file called 'startup.php' in
the root directory. Put the following code in the index.php file:

~~~php
# Startup tasks (define constants, etc)
require 'startup.php';
~~~

## Startup Tasks

The startup file is used to do some general startup tasks, like
defining constants, setting the error reporting level, etc. The first
part of our startup file looks like this:

~~~php
<?php
error_reporting(E_ALL);
if(version_compare(phpversion(), '7.0.0', '<') == true){die('PHP7.0 or newer'); }

// Constants:
define('DIRSEP', DIRECTORY_SEPARATOR);

// Get site path
$site_path = realpath(dirname(__FILE__) . DIRSEP) . DIRSEP;
define('site_path', $site_path);
~~~

In the above example we define some constants, get the site path,
and also check to make sure that the current PHP version is at least
7.0.

## The Registry

!!! note
    The registry design pattern is used here because it is very simple to
    understand. Today you would see more often a Service Manager or other
    similar design patterns.

The next thing we'll have to do is setup a Registry object to hold all
our global data. A registry object is passed around all the individual
objects in our MVC system, and is used to transport global data
through out the system without having to use the 'global' keyword
or $GLOBALS. Read "Using globals in PHP" for more information
on the registry object.

Add the following code to the startup.php file, below the code from
the previous example:

~~~php
$registry = new Registry;
~~~

If you now try to run the system, you should get the following error:

~~~
Fatal error:
Class 'Registry' not found in demo/startup.php on line 12
~~~

Not really a surprise, since we haven't created the Registry class yet,
nor have we included a file that contains the Registry class. We
could simply include it using the include() function, but let's use one
of PHP5's new features: __autoload().

The __autoload() magic function is used to dynamically load
classes. Whenever PHP encounters a non-existent class, it will first
call a custom __autoload() function, and only then declare an error.
This can be used to load classes on-the-fly.

Put one of the following codes before the previous code example:

~~~php
// For loading classes
function custom_autoload($class_name) {
	$filename = strtolower($class_name) . '.php';
	$file = site_path . 'classes' . DIRSEP . $filename;
	if(file_exists($file) == false){
		return false;
	}
	include($file);
}

spl_autoload_register('custom_autoload');
~~~

Our custom_autoload() function takes the class name, passed as an
argument, and checks if a similar named file exists in the classes
directory. If the file doesn't exist, the function will simply return
false and a fatal error will still show up, but if the file does exist, it
will be included, which means the class is suddenly there, and no
error is thrown. For it to work we need to define the
custom_autoload with sql_autoload_register(custom_autoload).

We haven't created the Registry class yet, which means we still get
an error, so let's do something about that.

## Creating the Registry class

The Registry class is used to pass global data around between the
individual objects, and is actually a really simple class, and needs no
more than three small methods.

First, create a new directory called 'classes', and then create a new
file called 'registry.php'. Put the following code in the registry.php
file:

~~~php
<?php
Class Registry {
	private $vars = array();
}
~~~

We now have a skeleton for the Registry class, and all we need to do
is add some methods. All the Registry class needs is a set() method,
to set data, and a get() method, to get data. Optionally we can also
add a remove() method to remove some data. The following three
methods will do, and should be added to the Registry class:

~~~php
function set($key, $var){
	if (isset($this->vars[$key]) == true) {
		throw new Exception('Unable to set var `' . $key . '`. Already set.');
	}
	$this->vars[$key] = $var;
	return true;
}

function get($key) {
	if(isset($this->vars[$key]) == false) {
		return null;
	}
	return $this->vars[$key];
}

function remove($var){
	unset($this->vars[$key]);
}
~~~

As you can see, these three methods are really basic, and all they do
is set, get and unset items from the $vars property. In the set()
method we also check if data with that particular key doesn't
already exist, and if it does, we throw an exception. This is to protect
from accidentally overwriting data.

We now have a fully functional Registry class, but we aren't going to
stop here. We're going to use one of SPL's features: ArrayAccess.
SPL, which is short for the Standard PHP Library, is a collection of
interfaces and classes that are meant to solve standard problems.

One of SPL's interfaces, ArrayAccess, can be used to give array
access to an object. Consider the following code snippet, do not
integrate it:

~~~php
<?php
$registry = new Registry;
// Set some data
$registry->set ('name', 'Dennis Pallett');
// Get data, using get()
echo $registry->get('name');
// Get data, using array access
echo $registry['name']
~~~

The array access makes it seem that $registry is an array, even
though it's an object. Although ArrayAccess has not real
advantages, it means less typing since you don't have to
continuously use ->get(). To use ArrayAccess you will first have to
change the first line of the class ("Class Registry"), which becomes:

~~~php
Class Registry implements ArrayAccess {
~~~

The Implements keyword is used to implement an interface, and
that's exactly what ArrayAccess is.

By implementing the ArrayAccess interface, the class must also add
four new methods:

~~~php
function offsetExists($offset) {
	return isset($this->vars[$offset]);
}

function offsetGet($offset) {
	return$this->get($offset);
}

function offsetSet($offset, $value) {
	$this->set($offset, $value);
}

function offsetUnset($offset) {
	unset($this->vars[$offset]);
}
~~~

These methods should be fairly self-explanatory, and more
information can be found in the SPL documentation.

Now that we have implemented ArrayAccess, we can use the object
just like an array, as you saw in the previous example and in the
following example:

~~~php
<?php
$registry = new Registry;

// Set some data
$registry->['name'] = 'MVC Tutorial';

// Get data, using get()
echo $registry->get('name');

// Get data, using array access
echo $registry['name']
~~~

Our Registry class is now finished, and if you try to run the system
everything should work (although nothing is displayed yet). Our
start up file is finished, and we can move to the next step of our
MVC system: setting up the database functionality, also called the
"Model".

## The Model

The 'M' or model part of the MVC system is responsible for
querying the database (or another external source) and providing
the data to the controller. We could have the appropriate model
loaded depending on the request, but I prefer to blur the lines
between the model and the controller, whereby the controller uses
a DB abstraction library to directly query the DB, instead of having a
separate model.

One thing we must do is add the code necessary to setup up a
connection with the database, and add it to our index page. There
are many great DB abstraction libraries available but PHP5 and higher comes with a great DB library
already - PDO - so there's no need for a different library.

Put the following code in the global.php in the root folder.

~~~php
<?php
$db = new PDO('mysql:host=localhost; dbname=demo', '[user]', '[password]');
$registry->set('db', $db);
~~~

Now we include the global.php in our index.php file in our root
folder.

~~~php
# Connect to DB
include 'global.php';
~~~

In the above example we first create a new instance of the PDO
library, and connect to our MySQL database. We then make the $db
global, by using our Registry class.

We won't need that for this tutorial bit it's important because we
would need it later if we want to get data from a database, like with
any other MVC System. The global.php should be diffrent from
system to system so that no sensitive data leaves you development
environment.

## Creating a sample model

Now if we want our program to use data that we deliver, we need to
implement a class like Member. This is our Model. The Model
represents the data that gets posted to the Controller.

In our example we want give out a member name. So we need our
model the possibility to hold information. For this we create a new
folder in our root directory called "models". Then we create a new
file called Model_Member.php in it.

~~~php
<?php
/**
* Model_Member
*/
class Model_Member {
	private $firstname;
	private $lastname;

	/**
	* __construct
	*
	* @param string $firstname
	* @param string $lastname
	* @return void
	*/
	public function __construct(string $firstname, string $lastname) {
		$this->firstname = $firstname;
		$this->lastname = $lastname;
	}

	/**
	* getFirstname
	*
	* @return String
	*/
	public function getFirstname() : String {
		return $this->firstname;
	}

	/**
	* getLastname
	*
	* @return String
	*/
	public function getLastname() : String {
		return $this->lastname;
	}

	/**
	* setFirstname
	*
	* @param string $firstname
	* @return void
	*/
	public function setFirstname(string $firstname) {
		$this->firstname = $firstname;
	}
	
	/**
	* setLastname
	*
	* @param string $lastname
	* @return void
	*/
	public function setLastname(string $lastname) {
		$this->lastname = $lastname;
	}
}
~~~

To give and receive information we use functions called getter and
setter. Like our getter getFirstname will return the String that was
saved if called. With our setters like setLastname we can give it a
String and it will be saved in the object.

!!! note "Some words about DocBlocks"
    As you might have noticed the code has PHP Doc Comment added.
    These comments add information about our functions in our code.
    We can later read it out and generate a code documentation if we
    want to. It describes what a function needs (@param) and what it
    might return (@return). Doc Comments always start with /** and
    end with */.

The model part of our system is pretty much finished for now, so
let's move on to the next part: writing the controller.
Writing the controller also means we will have to write a Router
class first. A Router class is responsible for loading the correct
controller, based on the request (the $router variable passed
through the URL). Let's write the Router class first.

## The Router class

Our Router class will have to analyse the request, and then load the
correct command/action (method) from the right controller. We will create a new Router.php file in our
classes folder. First step is to create a basic skeleton for the router
class:

~~~php
<?php
Class Router {
	private $registry;
	private $path;
	private $args = array();

	function __construct($registry){
		$this->registry = $registry;
	}
}
~~~

And then add the following lines to the index.php file:

~~~php
# Load router
$router = new Router($registry);
$registry->set('router', $router);
~~~

We've now added the Router class to our MVC system, but it
doesn't do anything yet, so let's add the necessary methods to the
Router class.

The first thing we will want to add is a setPath() method, which is
used to set the directory where we will holdall our controllers. The
setPath() method looks like this, and needs to be added to the
Router class:

~~~php
function setPath($path){
	$path = trim($path, '/\\');
	$path .= DIRSEP;
	
	if(is_dir($path) == false) {
		throw new Exception ('Invalid controller path: `' . $path . '`');
	}
	$this->path = $path;
}
~~~

!!! note
    When using a Unix filesystem change trim=($path, '/\\'); to trim=($path, '\\'); .

Then add the following line to the index.php file:

~~~php
$router->setPath(site_path . 'controllers');
~~~

Now that we've set the path to our controllers, we can write the
actual method responsible for loading the correct controller. This
method will be called delegate(), and will analyse the request. The
first bit of this method looks like this:

~~~php
function delegate(){
	// Analyse route
	$this->getController($file, $controller, $action, $args);
~~~

As you can see, it uses another method, getController() to get the
controller name, and a few other variables. This method looks like
this:

~~~php
/**
* getController
*
* @param mixed $file
* @param mixed $controller
* @param mixed $action
* @param mixed $args
* @return void
*/
private function getController(&$file, &$controller, &$action, &$args) {
	$route = (empty($_GET['route'])) ? '' : $_GET['route'];
	if(empty($route)){$route = 'index'; }
	
	// Get separate parts
	$route = trim($route, '\\');
	$parts = explode('/', $route);
	
	// Find right controller
	$cmd_path = $this->path;
	foreach($parts as $part){
		$fullpath = $cmd_path .'Controller_'. $part;
		
		// Is there a dir with this path?
		if(is_dir($fullpath)){
			$cmd_path .= $part . DIRSEP;
			array_shift($parts);
			continue;
		}
		
		// Find the file
		if(is_file($fullpath . '.php')){
			$controller = $part;
			array_shift($parts);
			break;
		}
	}
	if(empty($controller)){$controller = 'index';};
	
	// Get action
	$action = array_shift($parts);
	if(empty($action)){$action = 'index'; }
	$args = $parts;
	$file = $cmd_path . 'Controller_'. $controller . '.php';
	$this->registry->set('args', $args);
}
~~~

Let's go through this method. It first gets the value of the $route
query string variable, and then proceeds to split it into separate
parts, using the explode() function. If the request is 'members/view'
it would split it into array('members', 'view').

We then use a foreach loop to walk through each part, and first
check if the part is a directory. If it is, we add it to the filepath and
move to the next part. This allows us to put controllers in sub-
directories, and use hierarchies of controllers. If the part is not a
directory, but a file, we save it to the $controller variable, and exit
the loop since we've found the controller that we want.

After the loop we first make sure that a controller has been found,
and if there is no controller we use the default one called 'index'.

We then proceed to get the action that we need to execute. The
controller is a class that consists of several different methods, and
the action points to one of the methods. If no action is specified, we
use the default action called 'index'.

Lastly, we get the full file path of the controller by concatenating the
path, controller name and the extension.

Now that the request has been analysed it's up to the delegate()
method to load the controller and execute the action. The complete
delegate() method looks like this:

~~~php
function delegate() {
	// Analyze route
	$this->getController($file, $controller, $action, $args);
	
	// File available?
	if(is_readable($file) == false) {
		die('404 Not Found');
	}
	// Include the file
	include($file);
	
	// Initiate the class
	$class = 'Controller_' . $controller;
	$controller = new$class($this->registry);
	
	// Action available?
	if(is_callable(array($controller, $action)) == false) {
		die('404 Not Found');
	}
	
	// Run action
	$controller->$action();
}
~~~

After having analysed the request with the getController() method,
we first make sure that the file actually exists, and if it doesn't we
return an simple error message.
The next thing we do is include the controller file, and then initiate
the class, which should always be called Controller_[name]. We'll
learn more about the controller later on.

Then we check if the action exists and is executable by using the
is_callable() function. Lastly, we run the action, which completes
the role of the router.
Now that we have a fully working delegate() method, add the
following line to the index.php file:

~~~php
$router->delegate();
~~~

If you now try to run the system, you will either get the following
error, if you haven't yet created the 'controllers' directory:

~~~
Fatal error: Uncaught exception 'Exception' with message 'Invalid controller path:
`demo\controllers\` in demo\classes\router.php:18

Stack trace:
#0 \demo\index.php(13): Router->setPath('demo\...')
#1 {main} thrown in demo\classes\router.php on line 18
~~~

Or you will get the '404 Not Found' error, because there are no
controllers yet. But that's what we're going to create right now.

## The Controller

The controller part of our MVC system is actually very
simple, and requires very little work. First, make sure that
the 'controllers' directory exists. Then, create a new file
called 'controller_base.php' in the 'classes' directory, and
put the following code in it:

~~~php
<?php
Abstract Class Controller_Base {
	protected $registry;
	function __construct($registry){
		$this->registry = $registry;
	}
	
	abstract function index();
}
~~~

This abstract class will be the parent class for all our controllers, and
it does only two things: saves a copy of
the Registry class and makes sure that all our controllers have an
index() method.

Now let's create our first controller. Create a new file called
'Controller_Index.php' in the 'controllers' directory, and add the following code
to it:

~~~php
<?php
Class Controller_Index Extends Controller_Base {
	function index() {
		echo 'Hello from my MVC system';
	}
}
~~~

We've now created our first controller, and if you try to run our MVC
system now, you should see the following:

This means that our Router class did the job, and executed the
correct controller and action. Let's create another controller that
corresponds to a request that looks like 'members/view'. Create a
new file called 'controller_members.php' in a newly created
controllers directory, and add the following code to it:

~~~php
<?php
Class Controller_Members Extends Controller_Base {

	function index(){
		echo 'Default index of the `members` controllers';
	}
	
	function view() {
		echo 'You are viewing the members/view request';
	}
}
~~~

Now go to your MVC system, and make sure that the request is
'members/view', either by directly going there or by going to
index.php?route=members/view. You should get the following
result:

Just by creating a new controller class and adding a method we've
been able to define a whole new page in our MVC system, and we
didn't have to change anything else in our system. Our controllers
don't need to include a 'global.php' file or anything like it
whatsoever. Now that we've got the controller part working in our
MVC system, there's only one thing left: the 'V' or View part of our
MVC system.

## The View

Just like the Model, there are several different ways of doing the
View part of our MVC system. We could use the Router to
automatically load another file called something like
'view_{name}.php', but to keep this tutorial simple, we'll create a
custom View class, which can be used to show views.
First, create a new file called 'view.php' in the 'classes' directory,
and put the following code in it:

~~~php
<?php
Class View {
	private $registry;
	private $vars = array();

	function __construct($registry) {
		$this->registry = $registry;
	}
}
~~~

As you can see, we've now got the basic structure of our View class.
The next step is to add the following code to our index.php file,
before the Router statements:

~~~php
# Load view object
$view = new View($registry);
$registry->set('view', $view);
~~~

Because we want to use data from the model and controller in our
views, we will have to write a set() method to make variables
available in the view. See the example below:

~~~php
function set($varname, $value, $overwrite = false){
	if(isset($this->vars[$varname]) == true AND $overwrite == false) {
		trigger_error('Unable to set var `' . $varname . '`. Already set, and overwrite not
		allowed.',E_USER_NOTICE);
		return false;
	}
	$this->vars[$varname] = $value;
	return true;
}
function remove($varname) {
	unset($this->vars[$varname]);
	return true;
}
~~~

As you can see, the set() and remove() methods are fairly simple
methods, used to set and remove a variable.

Now that we can set variables, all we need to write is the show()
method, used for showing views. The easiest way is to create a
separate directory called 'views', which holds all our view files, and
then using an include() call to show a view. Of course your show()
method can be completely different, and load the views from the
database or do something else. See the code snippet below for the
show()method we'll be using:

~~~php
function show($name){
	$path = site_path . 'views' . DIRSEP . $name . '.php';
	if(file_exists($path) == false) {
		trigger_error('View `' . $name . '` does not exist.', E_USER_NOTICE);
		return false;
	}
	// Load variables
	foreach($this->vars as $key => $value) {
		$$key = $value;
	}
	include($path);
}
~~~

Our View class is now complete, and can be used to display views in
the controller. For example,create a new file called
'index_page.php' in the 'views' directory, and put the following
code in it:

~~~php
Hello from the View, <?php echo $first_name . " ". $last_name;?>!
~~~

Then, in the index controller (under controllers/index.php):

~~~php
function index() {
	$firstname = 'Peter';
	$lastname = 'Meier';
	
	$model_data = new Model_Member($firstname, $lastname);

	// Set the model into the registry
	$this->registry->set('member',$model_data);

	// Get some values from the model
	$this->registry['view']->set('first_name', $this->registry->get('member')->getFirstname());
	$this->registry['view']->set('last_name', $this->registry->get('member')->getLastname());

	// Render now the page to the user
	$this->registry['view']->show('index_page');
}
~~~

Here we set the parts of the name and then create a new object with
"new" and give it the parts of the name. Then we save our new
member object in our registy. After that we give the view the
information with set, so we set parameters for first and lastname
taken from our saved object in the registry.

!!! note
    Normally the data would
    come from a data base and not be written down in the controller,
    we wouldn't use the registry either, but for this tutorial we will use
    it this way just once as a demonstration.

If you now browse to our MVC system, you should get the following:
Now that we've got an active View component, our MVC system is
complete, and can be used to create a full-blown website. But there
are a few small things we have to take care of still. Also move the
router.php from the root folder to the classes folder.

## Security Measures

At the moment all the sub-directories, such as 'controllers' and
'views', are still publicly available to anyone who wants to visit it.
This could mean that users start running controllers or views that
should only be run by our system, so let's block access to those
directories.

With the .htaccess file this is really easy, and all it takes is the
following command:

~~~
Deny from ALL
~~~

Put the above command in a new file called .htaccess, and save this
file in the 'controllers' and 'views' directory (and any other directory
you want to protect). This will make sure that these directories are
completely off limits for everyone.
