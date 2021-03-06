Graphene tutorial           {#mainpage}
=================

## Requirements

This is what you need:

- PHP 5.3 or above (i.e. with namespace support)
- The mysqli driver
- Access to a MySql database
- Some way to run PHP scripts, either through a PHP enabled web server or via the command line interface

## Installation

Download and unzip Graphene. 

Copy the *graphene* directory to some directory where you want to create your PHP scripts.

If you use a web server to run your scripts, make sure the graphene directory is readable by the web server.

Graphene needs a directory it can write to, which is called the classpath. When you open a connection to Graphene you must specify as well the location of this directory. It is typically called *model* and placed aside from the *graphene* directory. 

If now you add a *helloworld.php* file where you can write your first script, this is how your script directory should now look like:

     
     /graphene
     /model
     helloworld.php
     
NOTE: The graphene directory contains some example file that can be recalled via HTTP. Especially if you edit the *init.php* in the examples to set real database connection parameters, you should avoid to deploy the *graphene/examples* directory to your production server, or at least make sure it is not reachable.
In general there is no reason for both the *model* and the *graphene* directory to be reachable via HTTP, so in a production environment it is a good idea to make them unavailable (how to do this depends on your web server).

## Including

That's easy: 

     include 'graphene/graphene.php';

## Connecting

    $db=graphene::open(array(
        "host"=>"localhost",
        "user"=>"dummy",
        "pwd"=>"dummy",
        "db"=>"test",
        "port"=>null,
        "prefix"=>"",
        "classpath"=>"./model"
    ));

Replace host, user, password and database name by those of an existing database you have access to. The port can be omitted or set to null, in which case the default MySql port (3306) will be used. The prefix also is optional and only useful if you want several Graphene databases in a single MySql database. The classpath is where Graphene should store its definition files and where it should search your custom classes, if any. We already made the *model* directory in the Installation section, so let's use it.

## Freezing and unfreezing

Graphene can work in two modes: *frozen* and *unfrozen*. The first one is good for production, while the second is very handy during development. It will allow you to create types and properties as you name them in your code. Graphene will try to infer some information from how you are using them and create the definition files in a directory called *definitions* inside the classpath you provided (in our case the *model* directory). 

You should periodically have a look at those files, modify them if you want to, throw away useless stuff and eventually freeze some property or the entire definition file when you're happy with it, so they will not be touched anymore even if Graphene is in *unfrozen* mode. 

Since we have nothing in the database, nor have we written any definition file, let's start unfreezing Graphene:

    $db->unfreeze();
    
By default Graphene is frozen, so in production you simply don't unfreeze it. If however you want to re-freeze it after having unfrozen, you can call:

     $db->freeze();

## Transactions

Before writing any data, we have to open a transaction.  

    $db->begin();

Being in a transaction allows you to write your data, having them reflected to the database while working, but avoiding conflicts with other concurrent write accesses. Furthermore it allows you to either commit (publish your changes to the database) or rollback on error (and leave the database untouched).

To commit / rollback use:

     $db->commit();
     $db->rollback();


## Writing data

Ok, now we're ready for the fun part. Just to summarize: your *helloworld.php* file should by now look somewhat like this:

     <?php
     
     include '../graphene.php';
     
     $db=graphene::open(array(
          "host"=>"localhost",
          "user"=>"dummy",
          "pwd"=>"dummy",
          "db"=>"test",
          "port"=>null,
          "prefix"=>"",
          "classpath"=>"./model"
     ));
     
     $db->begin();
     
We can avoid the *try / catch* block for the moment, we don't even want to commit at the end of the file. This is a handy way to run the script over and over again without filling your database with junk.

If you try to run the script, it will take some seconds since it has to create the Graphene tables in the targeted database. This will happen only the first time you open the connection.

Ok, let's go.

     $john=$db->Person();
     $john->firstName="John";
     $john->lastName="Smith";
    
     echo "John's first name is: ",$john->firstName,PHP_EOL;
    
     foreach( $db->select("Person#x and #x.firstName like 'J%'") as $person ) {
          echo "Found a person whose first name starts with 'J': ", 
               $person->firstName,' ',$person->lastName,PHP_EOL;
     }

And this should be the output:

     John's first name is: John
     Found a person whose first name starts with 'J': John Smith

If you are calling the script through the browser you'll probably want to surroud the whole thing by a &lt;PRE&gt; tag in order to get a decent output.

Let's add some more objects (or *nodes*) to our story:


     $bookshop=$db->Bookshop();
     $bookshop->owner=$john;
     $bookshop->name="John's bookshop";
     $bookshop->openSince=new DateTime("1986-05-13");
     
     $joyce=$db->Person(array("firstName"=>"James","lastName"=>"Joyce","isFamous"=>1));
     $fwake=$db->Book(array("title"=>"Finnegans wake","author"=>$joyce));
     
     $johnsbook=$db->Book(array("title"=>"How to run a bookshop","author"=>$john));
     
     $bookshop->books->add($fwake);
     $bookshop->books->add($johnsbook);


So... we have created two persons: John Smith and James Joyce. The first one is the proud owner of *John's bookshop* (since 1986!). The second one is a famous book writer and his celebrated novel "Finnegans wake" is sold in John's bookshop. This might have inspired John to write a book on his turn about how to run a bookshop, which is the only thing he really knows something about, and of course this one is also sold in his bookshop. 

Every property can be of one of the following data types:

- int 
- string
- float
- datetime (bound to PHP's DateTime class)
- node (properties that link from one node to another, also called *relations*)

Relations can be travelled as well the other way around by adding a '@' to the property name, or by assigning an alias in the definition files (see next section).

To get back all bookshops *owned by* John, for example, you can do:

     foreach( $john->get('@owner') as $bookshop ) { }


## Querying

Ok, now we can use our simple dataset to make some queries. This one is the simplest, and is not even a query:
     
     	$bookshop=$db->Bookshop->getBy("name","John's bookshop");

The *getBy* type function is very handy and you'll use it over and over again, but it is quite limited. To really query the database you use the *select* function:

     foreach ($db->select("Book#x and #x.title like 'f%'") as $book) {
          echo $book->title,PHP_EOL;
     }

What is called query is the string you pass to the *select* function. You can query the database, as we did in the above example, or directly a type:

     foreach ($db->Book->select("#x.title like 'f%'") as $book) {
          echo $book->title,PHP_EOL;
     }

In which case we don't have to tell *#x* is a book, since we are calling the Book type. 
You can even query a (node) property:

     foreach ($bookshop->books->select("title like 'f%'") as $book) {
          echo $book->title,PHP_EOL;
     }

By the way, #x can be omitted (as we did here) when it is the starting point of a relation or property.

A more interesting query is the following:

     Person#x and Book#book and Bookshop#bs=? and #bs.books=#book and #book.author=#x"

This can be translated into english as:

     Find all nodes #x such that: 
          #x is a Person
          and
          there is a Book #book 
          and 
          there is a Bookshop #bs being the first parameter we'll pass
          and
          #book is among the books of #bs
          and 
          the author of #book is #x

In other words: it will give you back all authors of any book sold in John's bookshop. The whole thing can be written in a much shorter way like this:

    $db->Person->select("@author.@books=?",$bookshop)
    
or, if you get confused by inverse properties, you can make a compromise:

    $db->Person->select("Bookshop#bs=? and #bs.books.author=#x",$bookshop);
    
In all cases, the query should return both John Smith and James Joyce in our example dataset.

And to finish the argument let's have a look at a quite complex query:

     Person#x and Bookshop#bookshop and #bookshop.owner=#x and #bookshop.books#book and 
     #book.author=#x and #bookshop.books#book2 and #book2.author.isFamous=1

In english:

     Find all nodes #x such that:
          #x is a Person
          and
          there is a Bookshop #bookshop
          and
          the owner of #bookshop is #x
          and
          there is a #book among #bookshop's books
          and 
          the author of #book is #x
          and
          there is also a #book2 among #bookshops books
          and the author of #book2 is famous
          
More briefly: find all persons that own a bookshop in which a book written by their owns is sold as well as at least one book written by a famous author.

Of course the result in our dataset will be again John Smith, since he meets all the conditions.

     
## The definition files

Now it is time to have a look to what has happened in the *model/definitions* directory. It should now contain following files:

     Book.def
     Bookshop.def
     Person.def

and possibly others if you made some more experiments in your *helloworld.php* file.

Let's go through the Person.def file, that should look somehow like this:

     ##### Person #####
     
     # AUTO-GENERATED
     string firstName
     
     # AUTO-GENERATED
     string lastName
     
     # AUTO-GENERATED
     Bookshop{} @owner
     
     # AUTO-GENERATED
     int isFamous
     
     # AUTO-GENERATED
     Book{} @author


This is what Graphene has understood so far about Persons. The lines starting with '#' are comments. The two curly brackets after Bookshop and Book indicate that it is a set and not a single valued property. Lists insted (where repetitions are allowed) are indicated by square brackets ('[]').

You can modify this file. For example we might decide that in our simple world each person can have at most one bookshop and change the line like this:

     Bookshop @owner as bookshop !

Now *@owner* has become a single valued property and, since we were editing it, we added as well a nice synonym (*bookshop*), so from now on to get the bookshop owned by a person we can simply do:

     $john->bookshop;

instead of:

     $john->get('@owner');
     
Furthermore we added an exclamation mark at the end of the line, which indicates the property is *frozen* and Graphene should not adapt its definition anymore (but rather throw an error if it is violated). 

Let's see what Graphene did in *Book.def*:

     ##### Book #####
     
     # AUTO-GENERATED
     string title required
     
     # AUTO-GENERATED
     Person author required
     
     # AUTO-GENERATED
     Bookshop{} @books

As you can notice, Graphene thinks the title and author are required fields. This is because every time we created a *Book* in our example these two properties have been passed as initialization arguments. Is it right or is he not? Up to you to decide... or leave it as it is and, if you happen to create a Book without any of these arguments, Graphene will relax the constraint unless you freeze.


You can also declare your type as being the subtype of another one (i.e. an extension), as in this one again picked up from the *UserManager* example:

     ##### User #####
     
     # My applications user class. It extends the user manager user by adding
     # some personal data.
     \supertype um_User
     
     # AUTO-GENERATED
     string firstName
     
     # AUTO-GENERATED
     string lastName
     
     # I added delete cascade, so the address is deleted along with the user.
     Adress address delete cascade 

## Customizing nodes

All nodes you get are normally instances of the *graphene\Node* class, unless you create your own class.
Try to create a file called *PersonNode.php* in the *model/classes* directory, and paste following code:

     <?php
     
     class PersonNode extends \graphene\Node 
     {
     
          function sayHello() 
          {
               echo "Hello!",PHP_EOL;
          }
          
     }


Now in your *helloworld.php* you can call:

     $john->sayHello();
     
And he will politely say "Hello!". Of course you can add more interesting functions to the *PersonNode* class, feel free to experiment.

Within the node class you can get the database connection by calling:

     $this->db();

As well as your node type by calling:

     $this->type();


## What's next

[Read the whole documentation](http://graphenelib.org/docs/annotated.html).

