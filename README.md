# MOA

[![Build Status](https://travis-ci.org/gajus/moa.png?branch=master)](https://travis-ci.org/gajus/moa)
[![Coverage Status](https://coveralls.io/repos/gajus/moa/badge.png?branch=master)](https://coveralls.io/r/gajus/moa?branch=master)
[![Latest Stable Version](https://poser.pugx.org/gajus/moa/version.png)](https://packagist.org/packages/gajus/moa)
[![License](https://poser.pugx.org/gajus/moa/license.png)](https://packagist.org/packages/gajus/moa)

MOA (Mother of All) is a database abstraction using [Active Record](http://en.wikipedia.org/wiki/Active_record_pattern) pattern:

> Active record is an approach to accessing data in a database. A database table or view is wrapped into a class. Thus, an object instance is tied to a single row in the table. After creation of an object, a new row is added to the table upon save. Any object loaded gets its information from the database. When an object is updated the corresponding row in the table is also updated. The wrapper class implements accessor methods or properties for each column in the table or view.

– http://en.wikipedia.org/wiki/Active_record_pattern

MOA is designed to handle [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations.

MOA is not [ORM](http://en.wikipedia.org/wiki/Object-relational_mapping). MOA does not work with object relations and dependencies. However, these libraries do:

* [Doctrine](http://www.doctrine-project.org/)
* [Propel](http://propelorm.org/)

MOA does not implement elaborate finders, filters or methods for querying data. However, these libraries do:

* [PHP ActiveRecord](https://github.com/jpfuentes2/php-activerecord)
* [Paris](https://github.com/j4mie/paris)
* [Parm](https://github.com/cassell/Parm)

## Hierarchy & Responsibilities

### Builder

MOA is using dynamic code generation to represent your database. [builder script](#builder-script) generates a file for each table using attributes fetched from the database (e.g. column name, type, default value, etc.). These classes are generated dynamically to reduce the amount of hand-coded duplication of the data representation.

This is an example of generated class:

```php
/**
 * This class is generated using https://github.com/gajus/moa.
 * Do not edit this file; it will be overwritten.
 */
abstract class Greedy extends \Gajus\MOA\Mother {
    const TABLE_NAME = 'greedy';
    const PRIMARY_KEY_NAME = 'id';

    static protected
        $columns = [
            'id' => [
                'column_type' => 'int(10) unsigned',
                'column_key' => 'PRI',
                'column_default' => NULL,
                'data_type' => 'int',
                'is_nullable' => false,
                'extra' => 'auto_increment',
                'character_maximum_length' => NULL,
            ],
            'name' => [
                'column_type' => 'varchar(100)',
                'column_key' => '',
                'column_default' => '',
                'data_type' => 'varchar',
                'is_nullable' => false,
                'extra' => '',
                'character_maximum_length' => 100,
            ],
        ];
}
```

With other Active Record implementations you do not need a generator because these properties are either hand-typed or fetched during the execution of the code. The former is tedious and error-prone, while the latter is a lazy-workaround that has a considerable performance hit.

### Mother

All models extend `Gajus\MOA\Mother`. Mother attempts to reduce the number of executions that would otherwise cause an error only at the time of interacting with the database. This is achived by using the prefetched table attributes to work out when:

* Accessing a non-existing property.
* Setting proprety that does not pass derived or custom validation logic.
* Saving object without all the required properties.

Furthemore, Mother is keeping track of all the changes made to the object instance. `UPDATE` query will include only the properties that have changed since the last synchronisation. If object is saved without changes, then `UPDATE` query is not executed.

> If you know a negative downside of the behaviour described above, please [contribute](https://github.com/gajus/moa/issues/1) a warning.

Delete operation will remove the object reference from the database and unset the primary key property value.

### Hierarchy

Using MOA you can [choose your own namespace](#builder-script)) and have your own [base class](#mother-1).

This is an example of an application hierarchy incorporating all of the MOA components:

```
Gajus\MOA\Mother
    you base model [optional]
        MOA generated models
            your hand-typed models [optional]
                your hand-typed domain logic [optional]
```

## API

This section of the documentation is using code examples to introduce you to the API.

### Create and Update

Object is inserted and updated with `save` method. Object is inserted to the database if instancy primary key property has no value. Otherwise, object is updated using the primary key property value.

```php
/**
 * @param PDO $db
 * @param mixed $data
 */
$person = new \My\App\Model\Person($db);

// Set property
$person['name'] = 'Foo';

// Insert object to the database
$person->save();

$person['name'] = 'Bar';

// Update object
$person->save();
```

### Delete Object

```php
$person->delete();
```

### Inflate Object

#### Inflate Object Using the Primary Key Value

```php
$person = new \My\App\Model\Person($db, 1);
```

Object data is retrieved from the database where primary key value is "1".

#### Inflate Object From Data

```php
$person = new \My\App\Model\Person($db, ['id' => 1, 'name' => 'Foo']);
```

This code assumes that you already have a record in the database with this primary key.

### Getters and Setters

MOA implements `ArrayAccess` interface. You can manipulate object properties using the array syntax, e.g.

```php
<?php
$car = new \My\App\Model\Car($db);
$car['colour'] = 'red';
$car->save();
```

If you need to set multiple properties at once:

```php
/**
 * Shorthand method to pass each array key, value pair to the setter.
 *
 * @param array $data
 * @return gajus\MOA\Mother
 */
public function populate (array $data);
```

## Naming Convention

MOA assumes that your models are writen using CamelCase convention (e.g. `MyTableName`). Table names must be singular (e.g. `Car` not `Cars`). MOA generated models will use CamelCase convention.

## Example

```php
<?php
$car = new \My\App\Model\Car($db); // $db is PDO instance
$car['colour'] = 'red';
$car->save();

echo $car['id']; // Newly entered record ID.
```

## Extending

### Mother

To inject logic between Mother and the generated models:

1. Extend `Gajus\MOA\Mother` class.
2. Build models using `--extends` property.

### Individual models

Models generated using MOA are `abstract`. You need to extend all models before you can use them:

```php
<?php
namespace My\App\Model;

class Car extends \Dynamically\Generated\Car {
    static public function getLastBought (\PDO $db) {
        $car = $db->query("SELECT `" . static::$properties['primary_key_name'] . "` FROM `" . static::$properties['table_name'] . "` ORDER BY `purchase_datetime` DESC LIMIT 1");
        
        return new static::__construct($db, $car[static::$properties['primary_key_name']]);
    }

    static public function getManyWhereColour (\PDO $db, $colour) {
        $sth = $db->prepare("SELECT * FROM `" . static::$properties['table_name'] . "` WHERE `colour` = ?");
        $sth->execute([ $colour ]);

        return $sth->fetchAll(\PDO::FETCH_ASSOC);
    }
}
```

> MOA convention is to prefix method names "getMany[Where]" for methods that return array and "get[Where]" that return an instance of `Mother`.

#### Triggers

|Name|Description|
|`afterInsert`|Triggered after `INSERT` but before the transaction is commited.|
|`afterUpdate`|Triggered after `UPDATE` but before the transaction is commited.|
|`afterDelete`|Triggered after `DELETE` but before the transaction is commited.|

Each of the above methods can interrupt the respective transaction.

## Builder Script

Models are built using `./bin/build.php`  script, e.g. unit testing dependencies in this repository are built using:

```bash
php ./bin/build.php --namespace "Sandbox\Model\MOA" --database "moa" --path "./tests/Sandbox/Model/MOA"
```

### Parameters

|Name|Description|
|---|---|
|`path`|[required] Path to the directory where the models will be created.|
|`database`|[required] MySQL database name.|
|`host`|MySQL database host.|
|`user`|MySQL database user.|
|`password`|MySQL database password.|
|`namespace`|[required] PHP class namespace;|
|`extends`|PHP class to extend. Defaults to "\Gajus\MOA\Mother".|

All `.php` files will be deleted from the destination `path`. The destination `path` must have an empty `.moa` file. This requirement is a measure to prevent accidental data loss.
