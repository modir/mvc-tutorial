# A simple Todo -App

Now that we have our own MVC up and running we can start
implementing functionality. We will create a Todo App to keep track
of our notes and tasks.

This Todo App can save new Notes, change existing ones and delete
the one we don't need anymore.

For ease of use we also change our naming convention.
So first of all we change our existing files. Everything in the
controllers folder gets the prefix "Controller_" i.e.:
Controller_Index.php and everything in models folder gets the
Prefix "Model_". Our classes in the "classes" folder stay as they are.

## Controller
To access this new part of our App we need to set up a new
controller. So when we open the the site with /Todolist at the end
of our URL our router can find it and the controller can spring into
action. The new controller is called Controller_Todolist.php and is
placed in the controllers folder. We add the following code:

~~~php
Class Controller_Todo Extends Controller_Base {
	function index(){
		$this->registry['view']->show('Todoentry_Page');
	}
}
~~~

## View
Our Controller now calls a view called "Todoitems_Page", but we
haven't had made this view yet so we have to make it. In the View
folder we create a new file called "Todoitems_Page".

Add the following line to the Todoitems_Page.php in our view
folder:

~~~
Hello from the new Todo view!
~~~

## Model

Now to integrate our Model. We need to consider what our model
shall hold in forms of data. We are making a todolist. We want to
know what we wrote down as a todo and we want to know if it's
done or not. So we need two variables, one is "$content" to hold
what we write down and the other is "$state" to check a task when
it's done. Of cause we also need $id, to handle saving the todoitem
into a database.

So we Create a new file called "Model_Todo.php" in our models
folder and add the following:

~~~php
<?php
class Model_Todoitem {
	private $id;
	private $content;
	private $state;

	/**
	* Get the value of id
	*/
	public function getId() {
		return $this->id;
	}
	
	/**
	* Set the value of id
	*
	* @return self
	*/
	public function setId($id) {
		$this->id = $id;
		return $this;
	}
	
	/**
	* Get the value of content
	*/
	public function getContent() {
		return $this->content;
	}
	
	/**
	* Set the value of content
	*
	* @return self
	*/
	public function setContent($content) {
		$this->content = $content;
		return $this;
	}
	
	/**
	* Get the value of state
	*/
	public function getState() {
		return $this->state;
	}
	
	/**
	* Set the value of state
	*
	* @return self
	*/
	public function setState($state) {
		$this->state = $state;
		return $this;
	}
}
~~~

As you see we don't have a constructor for this model. That is
because our PDO object will do it for us. When we create an object
from the database with the help of the PDO a constructor is call that
fills in the attributes automatically.

## Intregrating the Model into the Controller

If we want to use our model we need to expand upon our controller
to handle our requests. Then we need to build a repository to handle
the CRUD for App. We will also need a Repository wich we will do
after the Controller is done.

First we will expand the controller like this:

~~~php
Class Controller_Todolist extends Controller_Base {

	/**
	* index
	*
	* @return void
	*/
	function index(){
		$repo = new Repository_Todoitem($this->registry);
		$entries = $repo->fetchAll();
		$this->registry['view']->set('entries', $entries);
		$this->registry['view']->show('Todoitems_Page');
	}

	/**
	* show
	*
	* @return void
	*/
	function show() {
		$repo = new Repository_Todoitem($this->registry);
		$entry = $repo->fetch((int)$this->registry->get('args')[0]);
		$this->registry['view']->set('id', $entry->getId());
		$this->registry['view']->set('content', $entry->getContent());
		$this->registry['view']->set('state', $entry->getState());
		$this->registry['view']->show('Todoentry_Page');
	}
	
	/**
	* add
	*
	* @return void
	*/
	function add() {
		$this->registry['view']->show('Todoentry_Add_Page');
		if ($_SERVER['REQUEST_METHOD'] === 'POST'){
			$repo = new Repository_Todoitem($this->registry);
			//state checkbox from vie Returns NULL
			if($_POST['state'] == NULL){
				$_POST['state'] = 0;
			}
			if($repo->add($_POST)){
				header('Location: http://' .$_SERVER["HTTP_HOST"]. "/Todolist");
			}
		}
	}
	/**
	* delete
	*
	* @return void
	*/
	function delete() {
		$repo = new Repository_Todoitem($this->registry);
		if($repo->delete($_POST['id'])){
			header('Location: http://' .$_SERVER["HTTP_HOST"]. "/Todolist");
		}
	}
	/**
	* edit
	*
	* @return void
	*/
	function edit() {
		if ($_SERVER['REQUEST_METHOD'] === 'POST'){
			$repo = new Repository_Todoitem($this->registry);
			if($_POST['state'] == NULL){
				$_POST['state'] = 0;
			}
			if($repo->update($_POST)){
				header('Location: http://' .$_SERVER["HTTP_HOST"]. "/Todolist");
			}
		}
	}
}
~~~

Fist we expand our index function to get all todoentries with the
fetchall() function of our upcoming repository. Then we set our
entries into our view to be able to display them.

With our show function we want to get a single entry and display it
on a separate page page that shows only the entry. So we fetch one
entry with it's id and give the attributes that we want to display to
the view.

The add function will, as it says, add a new entry. For that we first
call for the page to add an entry. Now, if we get a POST request we
will set the state of our todoentry to 0 if we get NULL, because the
HTML checkbox we use for our state gives us NULL when
unchecked. Then we call add and give it our POST Request to fill in a
Todoitem in the database.

With delete we just give the id of our entry to the repository to find
and delete the entry.

The edit function is exactly like our add function with the only
difference that we call the update function of our repository.
Next up is our repository. Our repository lets us interact with our
database. We create, read, update and delete (CRUD) an entry with
the help of it.

As we did with our controller we also create a abstract Class called
"Repository_Base" in our classes folder:

~~~php
<?php
Abstract Class Repository_Base{
	protected $registry;
	
	/**
	* __construct
	*
	* @param mixed $registry
	* @return void
	*/
	function __construct($registry) {
		$this->registry = $registry;
		$this->db = $registry['db'];
	}
}
~~~

This class ensures that our repositories have access to the database
connection, our PDO, at any point.

Now create a new file called "Repository_Todoitem.php" in the
models folder and add the following code:

~~~php
<?php
Class Repository_Todoitem extends Repository_Base {
	/**
	* fetch
	*
	* @param int $id
	* @return object
	*/
	function fetch(int $id) : object {
		$query = 'SELECT `id`, `content`, `state` FROM `todo` WHERE `id` = :id';
		$statement = $this->db->prepare($query);
		$result = $statement->execute([':id' => $id]);
		if ($statement->rowCount() === 0) {
			throw new OutOfRangeException();
		}
		$todoItem = $statement->fetchObject('Model_Todoitem');
		return $todoItem;
	}
	
	/**
	* fetchAll
	*
	* @return array
	*/
	function fetchAll() : array {
		$query = 'SELECT `id`, `content`, `state` FROM `todo`';
		$statement = $this->db->prepare($query);
		$result = $statement->execute();
		$rows = $this->db->query($query)->fetchAll(PDO::FETCH_CLASS, 'Model_Todoitem');
		return $rows;
	}
	
	/**
	* add
	*
	* @param mixed $form
	* @return bool
	*/
	function add($form) : bool {
		$query = 'INSERT INTO `todo` SET `content` = :content, `state` = :state';
		$statement = $this->db->prepare($query);
		$result = $statement->execute([
			':content' => $form['content'],
			':state' => $form['state']
		]);
		return $result;
	}
	
	/**
	* delete
	*
	* @param mixed $id
	* @return bool
	*/
	function delete($id) : bool {
		$query = 'DELETE FROM `todo` WHERE `id` = :id';
		$statement = $this->db->prepare($query);
		$result = $statement->execute([':id' => $id]);
		return $result;
	}
	
	/**
	* update
	*
	* @param mixed $form
	* @return bool
	*/
	function update($form) : bool{
		$query = 'UPDATE `todo` SET `content` = :content, `state` = :state WHERE `id` = :id';
		$statement = $this->db->prepare($query);
		$result = $statement->execute([
			':id' => $form['id'],
			':content' => $form['content'],
			':state' => $form['state']
		]);
		return $result;
	}
}
~~~

All we do here is basically prepre our different statements for our
CRUD functionality. Than we either fetch our object or objects, fill
them in and return them. Or we provide data to the statements and
call upon the database to either update an entry or delete it with it's
id.

## Making a functional View

To now view our todos we need of cause the views to show them.
For that I prepared three views. "Todoitems_Page.php" to view all
of our entries. "Todoentry_Page.php" to view a single entry and edit
or delete it. And "Todoentry_Add_Page" for making a new entry.

Create these files in the views folder.

Todoitems_Page:

~~~php
<!DOCTYPE html>
<html lang="en">
<head>
<link rel="stylesheet" href="/styles.css">
<meta charset="UTF-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Todoitems</title>
</head>
<body>
<?php
foreach($entries as $entry) {
	if($entry->getState() == 1){
		$done = '- done';
	} else {
		$done = '- open';
	}
	echo
	'<a class="box" href="/Todolist/show/'.$entry->getId().'">'.$entry-
	>getContent().' '.$done.'</a>';
}
?>
<a href="/Todolist/add">Neuer Eintrag</a>
</body>
</html>
~~~

Todoentry_Page:

~~~php
<!DOCTYPE html>
<html lang="en">
<head>
<link rel="stylesheet" href="/styles.css">
<meta charset="UTF-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Todoentry</title>
</head>
<body>
<form method="POST" action="/Todolist/edit/" class="todoentry">
<textarea id="todotext" name="content" rows="4" cols="50" autofocus><?php
echo $content?></textarea>
<span> done?
<input type="checkbox" name="state" value=1 <?php if($state == 1) echo'
checked' ?> >
</span>
<span>
<input type="submit" value="Save">
<input type="hidden" name="id" value="<?=$id;?>">
</span>
</form>
<form method="POST" action="/Todolist/delete/">
<input type="submit" value="delete">
<input type="hidden" name="id" value="<?=$id;?>">
</form>
</body>
</html>
~~~

Todoentry_Add_Page:

~~~php
<!DOCTYPE html>
<html lang="en">
<head>
<link rel="stylesheet" href="/styles.css">
<meta charset="UTF-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>New Entry</title>
</head>
<body>
<h1>New Todo</h1>
<form method="POST">
<fieldset>
<label>
<span>
Content
<textarea id="todotext" name="content" rows="4"
cols="50" autofocus></textarea>
</span>
<br>
<span>
Status
<input type="checkbox" name="state" value=o>
</span>
</label>
</fieldset>
<input type="submit" value="Save">
</form>
</body>
</html>
~~~

Now we have every thing in place for the Todoapp to work.
