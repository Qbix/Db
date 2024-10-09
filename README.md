## 📊 Database Abstraction Layer

Qbix provides excellent facilities for using databases. There are several reasons to use them in your app, including:

- 🔒 Automatically sanitizing database queries to prevent SQL injection attacks. Without this, you might expose your site to serious security risks.
- ⚙️ It fires Qbix's events, allowing you to attach a hook wherever you may need it later, such as for logging database queries or to implement sharding.
- 🛠️ Writes correct SQL code for you and uses the PHP compiler to ensure correct syntax, even with arbitrary expressions.

Qbix also provides an <a href="#models">object-relational mapping layer</a>. Using this layer will let you use many kinds of databases for persistence, and your social apps can be designed to scale horizontally and handle millions of users out of the box.

For now, only MySQL is supported. If you'd like to contribute to the project, you are more than welcome to write an adapter for your favorite DBMS, such as PostgreSQL, SQLite, or Riak!

### 🔌 Connections

Your app can have one or more database connections, which you would normally set up in the `"Db"/"connections"` config field. Note that this information might be different for each development machine, and therefore connections should be specified in the `APP_DIR/local/app.json` config file. Here is an example:

```json
{
  "Db": {
    "connections": {
      "*": { /* common fields */
        "dsn": "mysql:host=localhost;dbname=YM",
        "username": "YouMixer",
        "password": "somePassword",
        "driver_options": {
          "3": 2
        }
      },
      "YouMixer": {
        "prefix": "ym_",
      },
      "Users": {
        "prefix": "users_",
      },
      "Streams": {
        "prefix": "streams_",
      }
    }
  }
}
```

Behind the scenes, Qbix's database abstraction layer uses [PDO](http://php.net/pdo), and all the connection information is used to establish a PDO connection (except the prefix, which is used to prefix table names). When adding a [plugin](QP/guide?page=modules) to the app, you'd typically have to add the database connections it uses (which are named in the plugin's `config/plugin.json` file under `"pluginInfo"/pluginName/"connections"`), and then run the [installer](QP/guide?page=installing) again.

To use a connection, you could write the following:

```php
// returns Db object for the YouMixer connection
$youmixer_db = Db::connect('YouMixer');
```

But the connection is not made until you actually execute the first query against the database. If you call `Db::connect` multiple times with the same connection name, it will just return the same object.

When adding a new database schema to an app, you are advised to pick a new connection name (typically named after the app itself) and add it to the `"Q"/"appInfo"/"connections"` array. Then you'll be able to write [scripts for the installer](QP/guide?page=scripts) to run when installing or updating the app.

### 🔍 Making Queries

Q's DB abstraction layer includes several classes, which are all found in the "Db" package. Among them:

- 🗄️ The `Db` class represents a database connection. Among other things, it is a wrapper for `PDO` objects.
- 🔍 The `Db_Query` class represents a database query (customized to the particular DBMS that the connection was made to) that you are in the process of building and then executing.
- 📊 The `Db_Result` class represents a result set from executing a query. Among other things, it is a wrapper for `PDOStatement` objects.
- 📄 The `Db_Row` class represents a row in the database. It is used by the ORM and is discussed in the article about [Classes and Models](QP/guide?page=models).

When you get a `Db` object, you can call methods on it, such as `select`, `insert`, `update`, and `delete`. They return an object of type `Db_Query`. That class, in turn, has its own methods, most of which also return a `Db_Query` object.

Here are some examples demonstrating a bunch of functionality at once:

```php
$db = Db::connect('YouMixer');

$q = $db->select('*', 'mix')
   ->where(array(
      'name' => 'My Mix', // 'My Mix' will be sanitized, and "=" will be prepended
      'by_user_id <' => 100, // Here, an extra "=" is not prepended
      'title LIKE ' => '%somethin%' // No "=" is prepended due to the space after "LIKE".
   ))->orderBy('songCount', false)
   ->limit(5, 1); // you can chain these as much as you need

// add more things to the query at any time
$q2 = $q->orWhere('id < 3');

$q3 = $db->update('mix')
   ->set(array(
     'a' => 'b',
     'c' => 'd'
   ))->where(array(
     'a' => 5
   ));

$q4 = $db->delete('mix')
   ->where(array('id' => $mix_id)); // $mix_id will be sanitized

$q5 = $db->insert('mix', compact('name', 'by_user_id'));

$q6 = $db->rawQuery(
  "SELECT name, by_user_id FROM mix"
);
```

### 🚀 Executing Queries

There are a couple of ways you can execute a query and fetch the results. One way is to get a `Db_Result` object and then fetch:

```php
$r = $q->execute();
$r->fetchAll(PDO::FETCH_ASSOC);

// or all in one line:
Db::connect('YouMixer')
   ->select('*', 'mix')
   ->where('id > 5') // you can pass strings here
   ->execute()->fetchAll();
```

A second way involves calling "fetch"-type methods directly on a query:

```php
// just fetch an array
$q->fetchAll(PDO::FETCH_ASSOC);

// or all in one line:
Db::connect('YouMixer')
   ->select('*', 'mix')
   ->where('id > 5')
   ->fetchAll(PDO::FETCH_BOTH);
```

The second way implicitly executes the query (and obtains a Db_Result) before fetching. Besides being shorter, the second way makes use of caching based on the query's SQL content. That means, if you use it twice in the same PHP script, it will only hit the database once, having cached the results.

An actual (PDO) connection is made to the database only when the first query is executed against that connection. You can also hook the "Db/query/execute" event for your own needs. For example, Qbix does this in order to implement [sharding](http://en.wikipedia.org/wiki/Shard_(database_architecture)) in the application layer.

### 📜 Lists of Values

You can specify lists of values as an array in your `where()` clauses, as follows:

```php
Db::connect('Users')
   ->select('*', 'users_vote')
   ->where(array(
     'userId' => 'tlnoybda',
     'forType' => array('type1', 'type2', 'type3')
   ));
```

The SQL that is generated looks like this:

```sql
SELECT * FROM users_vote
WHERE userId = 'tlnoybda'
AND forType IN ('type1', 'type2', 'type3')
```

### 🎯 Vector Value Lists

Sometimes you want to test several values at once, as a vector. Qbix supports this as well, although in this case, this causes [database sharding](QP/guide?page=sharding) to map such a query to run on every shard.

```php
Db::connect('Users')
   ->select('*', 'streams_stream')
   ->where(array(
     'publisherId, name' => array(
       array('tlnoybda', 'firstName'),
       array('tlnoybda', 'lastName'),
       array('uqoeicuz', 'firstName'),
       array('uqoeicuz', 'lastName')
     ),
     'something' => 'else'
   ));
```

The SQL that is generated looks like this:

```sql
SELECT * FROM streams_stream
WHERE (publisherId, streamName) IN (
  ('tlnoybda', 'firstName'),
  ('tlnoybda', 'lastName'),
  ('uqoeicuz', 'firstName'),
  ('uqoeicuz', 'lastName')
) AND something = 'else'
```

### 📈 Database Ranges

Instead of exact values, you can specify ranges in your `where()` clauses, as follows:

```php
$range = new Db_Range(
  $min, $includeMin, $includeMax, $max
);
Db::connect('Users')
   ->select('*', 'vote')
   ->where(array(
     'forType' => 'article',
     'weightTotal' => $range
   ));
```

This will produce the appropriate inequalities when composing the database query. It also works with [database sharding](QP/guide?page=sharding).

### ✨ Database Expressions

By default, Qbix's database library sanitizes values that you pass when building queries. For example, if you wrote:

```php
$results = $db->select('*', 'mix')
   ->where('time_created >' => 
     "CURRENT_TIMESTAMP - INTERVAL 1 DAY"
   )->fetchAll(PDO::FETCH_ASSOC);
```

Qbix would treat `"CURRENT_TIMESTAMP - INTERVAL 1 DAY"` as a string and sanitize it, which would not give the intended result. To pass a raw expression to the database, use `Db_Expression`:

```php
$results = $db->select('*', 'mix')
   ->where('time_created >', new Db_Expression("CURRENT_TIMESTAMP - INTERVAL 1 DAY"))
   ->fetchAll(PDO::FETCH_ASSOC);
```

This allows you to insert raw SQL expressions directly into your queries without being sanitized as strings. 🛡️

### 📊 Aggregates and Grouping

Qbix also supports SQL aggregate functions such as `COUNT()`, `SUM()`, and `AVG()`, as well as grouping results. Here’s an example that counts the number of mixes by each user:

```php
$results = $db->select(array('by_user_id', new Db_Expression('COUNT(*) AS num_mixes')), 'mix')
   ->groupBy('by_user_id')
   ->fetchAll(PDO::FETCH_ASSOC);
```

This query selects the `by_user_id` and counts the number of mixes for each user, grouping by the `by_user_id`. 📈

### 🔗 Joins

To retrieve related data from multiple tables, you can use `join` methods:

```php
$results = $db->select('*', 'mix')
   ->join('user', 'mix.by_user_id = user.id')
   ->where('mix.id >', 5)
   ->fetchAll(PDO::FETCH_ASSOC);
```

This query selects all fields from the `mix` table and joins the `user` table based on the condition that the `by_user_id` in `mix` matches the `id` in the `user` table.

You can also use different types of joins (`INNER JOIN`, `LEFT JOIN`, etc.) by specifying the type of join in the method call:

```php
$results = $db->select('*', 'mix')
   ->leftJoin('user', 'mix.by_user_id = user.id')
   ->where('mix.id >', 5)
   ->fetchAll(PDO::FETCH_ASSOC);
```

### 💼 Transactions

Qbix allows you to manage database transactions, providing methods to `beginTransaction()`, `commit()`, and `rollBack()` when working with a database connection:

```php
$db->beginTransaction();

try {
    $db->insert('mix', array('name' => 'New Mix', 'by_user_id' => 1));
    $db->update('user', array('mix_count' => new Db_Expression('mix_count + 1')))
       ->where('id', 1);
    $db->commit();
} catch (Exception $e) {
    $db->rollBack();
    throw $e;
}
```

This ensures that if any part of the transaction fails, the changes made in the database will be rolled back. 🔄

### 🗄️ Caching

Qbix's database abstraction layer supports query caching to reduce the load on the database server. By using a cache, frequently executed queries will only hit the database once and return cached results on subsequent requests.

```php
$results = $db->select('*', 'mix')
   ->where('id >', 5)
   ->cache(60) // cache the result for 60 seconds
   ->fetchAll(PDO::FETCH_ASSOC);
```

In this example, the query result will be cached for 60 seconds, so the next time the query is run within that time period, the results will be served from the cache instead of querying the database again. ⏳

### 🐞 Debugging

To help with debugging, Qbix's database abstraction layer provides a way to log and view queries executed by the database:

```php
$db->enableLogging();
$results = $db->select('*', 'mix')
   ->where('id >', 5)
   ->fetchAll(PDO::FETCH_ASSOC);

$log = $db->getQueryLog();
print_r($log);
```

By enabling logging, you can see the queries executed, which is useful for debugging and optimizing performance. 🔍

---

## 📚 Models

### 📦 Classes

Using PHP's autoload mechanism, Qbix can locate files where classes are defined and load them when you first invoke them in your code. For a class named `Foo_Bar_Baz`, it will try to load the `classes/Foo/Bar/Baz.php` file. This convention is similar to PEAR and the [Zend Framework](http://framework.zend.com). In fact, you can take classes from both of those and simply drop them into the `classes` folder, and Qbix will autoload them when you need them. This is one way that Qbix lets you use lots of cool classes from other libraries and frameworks without reinventing the wheel. 🎉

### 🛠️ Generating Models

We have spent a lot of time talking about views and controllers in other articles in the guide. In this one, we will focus on the M part of MVC—the models. Whereas the C part is usually implemented using `handlers`, the M part is implemented using `classes`.

In Qbix, a model is a class that represents a persistent object, such as a database row (or table), or contains functionality to manipulate these objects. You can actually generate models automatically from existing tables after you have set up a database connection in the app's config, simply by running the script `APP_DIR/scripts/Q/models.php`.

Yes, it's as simple as that. You can see the files generated for the models in the `classes` folder. One of them is named after the database connection and has methods like `Something::db()`. The others are prefixed with `Something_`, and represent the tables in the database. 📂

Once the files for the models are generated, you can edit them. Don't edit the ones inside the `classes/Base/` folder, since your changes will be overwritten the next time you decide to re-generate the schema. However, you are free to edit the non-base model classes, and you can implement any methods you wish in your models by adding them to these classes. 🛠️

### 📑 The Db_Row Class

Qbix comes with a built-in `Db_Row` class, which contains common functionality that models have. In fact, the models that Qbix autogenerates for you all extend this class. Out of the box, `Db_Row` is a full ORM (object-relational mapper) that implements the ActiveRecord pattern. In addition to PDO methods like `fetchAll`, you can fetch and fill `Db_Row` objects, like so:

```php
$rows = YouMixer_Mix::select('*')
          ->fetchDbRows();
```

Although you can create instances of `Db_Row` directly, you are not expected to do that. Instead, `Db_Row` is meant to be extended by a class implementing a model. For example, an autogenerated model class like `Users_User` would extend `Base_Users_User`, which in turn extends `Db_Row`, thereby inheriting all of its methods. Here are some examples of how you can use them:

```php
// Inserting new rows:

$user = new Users_User();
$user->username = "SomeoneCool";
$user->save(); // insert into db
$user_id = $user->id; // id was autogenerated

// Retrieving rows:

$user = new User_User();
$user->username = 'Gregory';
$user = $user->retrieve();
// SELECT * FROM myDatabase.myPrefix_user
// WHERE username = "Gregory"

if (!$user) {
  throw new Exception("No such user");
}
// the user row has been retrieved
echo $user->username; // 🎉

// Retrieving a bunch of rows:

$rows = Users_User::select('*')
        ->where('id = 4')
        ->fetchDbRows();
// SELECT * FROM myDatabase.myPrefix_user
// WHERE id = 4

// Updating rows one at a time
// For mass updates, skip the ORM and
// use Users_User::update() instead.

$r = $user->retrieve();
$user->content = 'Gregory';
$user->save(); // issues an INSERT or UPDATE query,
// depending on $r

// Deleting rows

$user = new Users_User();
$user->id = 4;
$user->remove(); // 🗑️

```

Every `Db_Row` object also includes a `Q_Tree`, which means it supports the following methods, used to associate additional app data with the row, without saving it to the database:

```php
$fields = $row->getAll();
$foo = $row->get('foo', 'bar', $default);
$row->set('foo', 'bar', $value);
$row->clear('foo', 'bar');
```

Besides this, `Db_Row` has a lot more functionality, which you can discover in the class reference. 📖

---

### 📤 Exporting Data to the Client

Often, when generating a page with PHP, you may want to output some data from `Db_Row` objects to the client JS environment. Here is how you would typically do that:

```php
Q_Response::setScriptData(
  "First.someRow", 
  $row->exportArray($options)
);

Q_Response::setScriptData(
  "First.manyRows", 
  Db::exportArray($rows, $options)
);
```

Each Model class can override the `exportArray($options)` method to return an array that is safe to send to the client. 📡

### 🔄 What is Autogenerated

When you autogenerate models from a particular database, Qbix creates classes named something like `Base_ConnName_TableName`. These classes already do a lot of things for you, including:

- **🗃️ Information** — The model overrides the `setUp` method of `Db_Row` and specifies the names of the table and database connection, as well as the fields of the primary key for the table. This information is used by `Db_Row` when it generates queries to execute.
- **✔️ Validation** — Before a value is assigned to a specific field, the model checks that it fits the type of the field (column) as it is described in the schema.
- **🔍 Enumeration** — You can get a list of all the field names by calling `::fieldNames()`.
- **⏰ Magic fields** — If your table contains fields called `"insertedTime"` and `"updatedTime"` (of type `datetime`), they are filled with the right value when you call `$row->save()`.
- **✨ Helper methods** — When you call methods such as `Users_User::select()`, `Users_User::update()`, and so on, a query is returned that automatically fills in your database and table name. To illustrate:

```php
$users = Users_User::select('*')
  ->where(array(
    'name LIKE ' => $pattern
  ))->fetchDbRows();
	
// Now, $users is an array of
// zero or more Users_User objects. 🌟
```

### 🔗 Relations

One of the things you will want to add to your models is the relationships between your tables. You can, of course, write your own methods, such as `$user->getArticles()`. However, you can also tell Qbix's database library about these relationships and have it generate the code for you. 📚

The place to set this up is your model's `setUp()` method, where you can fill in your own code. Here, you would use the `$this->hasOne(...)` and `$this->hasMany(...)` methods to tell Qbix about the relationships between your models. Below is an example:

```php
class Items_Article extends Base_Items_Article
{
  function setUp()
  {
    parent::setUp();
    
    $this->hasOne('author', array(
      'a' => 'Items_Article',
      'u' => 'Users_User'             
      // returns Users_User objects
    ), array('a.byUserId' => 'u.id'));
    
    $this->hasMany('tags', array(
      'a' => 'Items_Article',
      't' => 'Items_Tag'              
      // returns Items_Tag objects
    ), array('a.itemId' => 't.itemId'));
    
    $this->hasMany('tagVotes', 
      array(
        'a' => 'Items_Article',
        'u' => 'Users_User',
        'tv' => 'Items_TagVote',    
        // returns Items_TagVote objects
      ),
      array('a.itemId' => 'tv.itemId'),
      array('u.id' => 'tv.byUserId')
    );
  }
}
```

You can then retrieve related objects by doing things like:

```php
$article->get_tags('*');
$article->get_tagVotes('*', array(
  'u' => $user // get votes placed by user
));
$article->get_author('u.firstName, u.lastName');
```

### 💾 Caching

As we saw in the [database article](QP/guide {"page": "database"}), Qbix does caching at the level of query execution by default (i.e., the same SQL only hits the database once). 🔄 However, Qbix also encourages caching at the level of models. To help you do this, each `Db_Row` object supports the methods `get`, `set`, and `clear`, which let you set arbitrary data on the rows. This data is used only in the script and isn't saved back to the database:

```php
// Get related tags
$tags = $art->get_tags(); 🏷️

// Save the result in the cache
$art->set('tags', $tags); 💾

// Sometime later:
// Check the cache -- if not there,
// then retrieve tags
$tags = $art->get('tags', $art->get_tags()); 🔍
```
