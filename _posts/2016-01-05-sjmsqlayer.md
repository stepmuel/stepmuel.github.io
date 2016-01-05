---
layout: post
title: "Clean SQL in PHP made easy"
date: 2016-01-05 03:57:22 +0100
tags: php coding
---

Over the years I have written many PHP applications that require SQL databases of some sort. I often reused my code, and developed some handy helpers over time. My current go-to solution reduces my development times massively and I want to share it with the world. 

# Problem

Executing an SQL request in PHP is usually done using PHP Data Objects (PDO). This abstraction layer allows to write code which is more or less independent of the database used. Nevertheless, the whole concept of using SQL inside PHP seems somewhat archaic. Similar to regular expressions, using a different syntax or even language for a very specialized task can sometimes be justified. But what if the query has to be dynamic? We basically write code to create a string for a different code (the SQL query). At some point, database access starts to look a lot like compiler design. 

Most SQL queries I use contain very simple `INSERT`, `UPDATE`, `SELECT` or `DELETE` statements. The code to create the required queries can easily be wrapped in convenient functions. Ideally, those functions should handle escaping values, do basic error handling and provide some flexibility like allowing to add `ORDER BY` or `LIMIT`. 

# Security

PHP has a long tradition of being considered an insecure language. That sentiment resulted mostly from earlier versions allowing to overwrite global variables by modifying the request query by default. Ease of development attracted many newcomers with little security awareness, which led to many vulnerable PHP applications. Another very common attack vector is SQL code injection, enabled by not properly handling user input. 

While it is certainly possible to write secure code in PHP, the language doesn't always make it easy. When creating quick prototypes, even experienced developers sometimes refrain from escaping all their variables, just to keep the code short and readable. But bad prototype code can still somehow end up in productive systems. So how can we get other developers and ourselves to write secure code from the beginning? By making it easier than the insecure way!

# Creating SQL from Dictionaries

Let's start with with a simple table in SQLite.

{% highlight sql %}
CREATE TABLE books (
  id INTEGER PRIMARY KEY,
  title TEXT,
  published INTEGER,
  pages INTEGER
);
{% endhighlight %}

Relational databases map very well to arrays of dictionaries. By using the `PDO::FETCH_ASSOC` fetch style, a row from the table above might look like this:

{% highlight json %}
{
  "id": 3,
  "title": "Don Quixote",
  "published": 1605,
  "pages": 992
}
{% endhighlight %}

But how do we insert such a row into the table? The SQL we want is simple. But why do we have to write it at all, when we already have the dictionary with all the information? Many `WHERE` statements have the form `key1=val1 AND key2=val2 AND etc.` which can be represented by a dictionary equally well. 

{% highlight sql %}
INSERT INTO books (id, title, published, pages) 
   VALUES (3, "Don Quixote", 1605, 992);
UPDATE books
   SET name = "Old and Huge", pages = 9001
   WHERE published = 1605 AND pages = 992;
SELECT name FROM books WHERE id = 3;
DELETE FROM books WHERE published = 1605;
{% endhighlight %}

My desire to use dictionaries for database access led to my first SQL API, which I named SJMTable.

{% highlight php startinline %}
class SJMTable {
    public function __construct($pdo, $table);
    public function insert($data, $args=array());
    public function update($where, $data, $args=array());
    public function select($keys, $where, $args=array());
    public function delete($where, $args=array());
}
{% endhighlight %}

The constructor needs a PDO object connected to a database and the name of a table. `$data` contains data to insert or update, and `$where` is used to create a `WHERE` statement by connecting the entries with `AND` (key1 = value1 AND key2 = value2 etc.). `$args` allows to optionally add order keys, limits and offsets, or simply append some SQL to the end of the query. The select function returns an array containing the fetched rows, insert returns the primary key of the new row and the other functions return the number of affected rows. 

# A New Syntax

The simple API covers a surprisingly large amount of SQL scenarios. The generic `$args` argument adds some flexibility, but the solution seems a bit "hacky" and still doesn't cover all possibilities. Especially joins would be very complex to add support for. Instead of adding all this complexity to the API, we can solve all our problems by introducing a single function that works similar to `printf` and a format syntax to replicate the previous functionality. 

{% highlight php startinline %}
// Syntax:
// %@ -> quoted value/list
// %K -> unquoted value/list
// %W -> where (WHERE %W), connected with 'AND'
// %S -> assign (for UPDATE SET %A)
// %I -> insert (INSERT INTO %K %I)
$dbl = new SJMSQLayer($pdo);
$dbl->query('INSERT INTO books %I', $data);
$dbl->query('UPDATE foo SET %S WHERE %W', $data, $where);
$dbl->query('SELECT name FROM books WHERE %W');
$dbl->query('DELETE FROM books WHERE %W', $where);
{% endhighlight %}

The resulting code is compact and very easy to read for everyone who knows SQL. `%W`, `%S` and `%I` provide the same conveniences as the previous API and `%@` and `%K` allow to insert additional quoted or unquoted variables. When using an array as argument, things like `WHERE id IN (%@)` lead to valid and perfectly quoted SQL. 

# Method Chaining

The query function returns a SJMSQLayerStatement object. That object handles a couple of common things you might want to do with SQL queries or allows you to simply get the generated SQL string.

{% highlight php startinline %}
class SJMSQLayerStatement {
	public $sql = null;
	public function exec();
	public function get($key=false);
	public function getAll($key=false);
	public function getDict($dictKey, $valueKey=false);
	public function getGroup($groupKey, $valueKey=false);
	public function lastInsertId();
}
{% endhighlight %}

By using method chaining, the functionality of SJMTable can still be achieved by single lines.

{% highlight php startinline %}
$dbl->query("DROP TABLE %K", "books")->exec();
{% endhighlight %}

`get` returns a single row from a `SELECT` query, `getAll` returns an array with all rows. `getDict` will also return all rows, but instead of an array, the result will be a dictionary where each row is addressed by its value of `$dictKey`. `getGroup` is used if the value by which the rows are addressed is not unique. It will return a dictionary of arrays. Using the optional parameters `$key` and `$valueKey` will fetch single values instead of the whole rows. 

{% highlight php startinline %}
$stm = $dbl->query("SELECT * FROM books");
// get array of all book id's
$stm->getAll("id");
// get a dictionary of book titles by id
$stm->getDict("id", "title");
// get a dictionary of book id's by year
$stm->getDict("published", "id");
{% endhighlight %}

# Source Code

The source code is available on [GitHub](https://github.com/stepmuel/SJMSQLayer). The SJMSQLayer class is very small (less than 100 lines); partially due to the lack of inline documentation. For now, just consider it an example implementation on how to improve your SQL experience. I might add some proper documentation at a later time, especially if people other than me want to use it. I'm also not sure which license I should use for it. I'd be happy to get some recommendations! 

If my code saves you a significant amount of time, consider a small donation to my [PayPal account](https://www.paypal.me/heap).







