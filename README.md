# GDPR SQL dump

A drop-in replacement for `mysqldump` or `mariadb` commands that optionally sanitizes DB fields for better GDPR conformity. It can be integrated with other tools such as `drush` or `backup_migrate`.

It is based on the [eaudeweb/mysqldump\-php](https://github.com/eaudeweb/mysqldump-php) forked from [ifsnop/mysqldump\-php](https://github.com/ifsnop/mysqldump-php) library, and can in principle dump any database supported by PDO.

It is using [Faker](https://packagist.org/packages/fzaninotto/faker) to populate columns with sanitized (random) data. 

## How to use

There are currently two ways to manipulate data:
1. by manipulating the actual SQL queries that run on the server (using the `gdpr-expressions` path)
2. by replacing column output before the dump is generated (using the `gdpr-replacements` or the `gdpr-replacements-file` option)

### 1. Using `gdpr-expressions`

The fields to obfuscate are passed via a `--gdpr-expressions` parameter. Note that we use `uid` expression to satisfy unique keys.

Without obfuscation:

```
$ ../vendor/bin/mysqldump drupal --host=mariadb --user=drupal --password=xxxxxxxx users_field_data --debug-sql
...
--
-- Dumping data for table `users_field_data`
--

/* SELECT `uid`,`langcode`,`preferred_langcode`,`preferred_admin_langcode`,`name`,`pass`,`mail`,`timezone`,`status`,`created`,`changed`,`access`,`login`,`init`,`default_langcode` FROM `users_field_data` */

INSERT INTO `users_field_data` VALUES (0,'en','en',NULL,'',NULL,NULL,'',0,1523397207,1523397207,0,0,NULL,1);
INSERT INTO `users_field_data` VALUES (1,'en','en',NULL,'admin','$S$Eb6kZl.9OFjoa69Z05pzUhaZJ6vpKaGZVpnjAxxLJ7ip0zOwanEV','admin@example.com','UTC',1,1523397207,1523397207,0,0,'admin@example.com',1);
```

With obfuscation:

```
# Example to dump individual anonymized SQL table
$ ./vendor/bin/mysqldump DATABASE --host=mariadb --user=drupal --password=xxxxxxxx users_field_data --gdpr-expressions='{"users_field_data":{"name":"uid","mail":"uid","pass":"\"\""}}' --debug-sql
...
--
-- Dumping data for table `users_field_data`
--

/* SELECT `uid`,`langcode`,`preferred_langcode`,`preferred_admin_langcode`,uid as name,"" as pass,uid as mail,`timezone`,`status`,`created`,`changed`,`access`,`login`,uid as init,`default_langcode` FROM `users_field_data` */

INSERT INTO `users_field_data` VALUES (0,'en','en',NULL,'0','','0','',0,1523397207,1523397207,0,0,'0',1);
INSERT INTO `users_field_data` VALUES (1,'en','en',NULL,'1','','1','UTC',1,1523397207,1523397207,0,0,'1',1);
```

### Using `gdpr-replacements` or `gdpr-replacements-file`

The tool is looking either for the `gdpr-replacements` or the `gdpr-replacements-file` option, passed as a command line argument, or as part of `my.cnf` file.

The argument to the `gdpr-replacements` command is a JSON string, while for `gdpr-replacements-file` is a path to a JSON file. For both, the format of the JSON is:

```JSON
{
    "tableName" : {
        "columnName1": {
            "formatter": "formatterType",
            "unique": true,
            "arguments": "....",
            "exclude": {},
            "include": {}
        },
        "columnName2": {
            "formatter": "formatterType"
        }
    }
}
```

* **formatterType** supports one of the following values:
    * **name** - random person name
    * **phoneNumber** - random phone number
    * **username** - random username
    * **password** - random password
    * **email** - random email address
    * **date** - random date
    * **longText** - random long piece of text
    * **number** - random number
    * **randomText** - random sentence
    * **text** - generates a paragraph of text
    * **uri** - generates an URI
    * **clear** - replaces with an empty string
* **unique** - Generates unique values that do not repeat, useful for primary keys / unique indexes.
* **arguments** - A set of arguments to pass to the Faker call, for example `passthrough` with `["613a303a7b7d"]` sets this exact value.
* **exclude** - Excludes rows from anonymization based on the passed criteria, examples:
    * Exclude using an SQL query: `"exclude": { "uid": "SELECT uid FROM users_field_data WHERE mail LIKE '%@example.org'" }`, where `uid` is a column in the same table
    * Exclude based on field value:   `"exclude": { "uid": [ 0, 1 ] }`
* **include** - Anonymize only rows matching criteria, values are similar to `exclude`. Ignored when also `exclude` is present

During an SQL dump, based on this configuration is going to replace the values in each table cell.

#### MySQL options file

You are able to have your `gdpr-expressions` declared in a mysql options file such as `~/.my.cnf` or `/etc/my.cnf` under the `[mysqldump]` section. See [MySQL/MariaDB options file](https://dev.mysql.com/doc/refman/8.0/en/option-files.html).

Example:

```
[mysqldump]
gdpr-replacements='{"fakertest":{"name": {"formatter":"name"}, "telephone": {"formatter":"phoneNumber"}}}'

```

## Use in Drupal projects (with Drush)

1. Add this repository to your `composer.json`:
```JSON
{
    "name": "gdpr-dump",
    "type": "vcs",
    "url": "https://github.com/eaudeweb/gdpr-dump",
    "no-api": true
},
```
2. Install the library in your project: `composer require eaudeweb/gdpr-dump`
3. Create `anonymize.schema.json` JSON file to anonymize user data
4. Optionally, use the [SQL Dump action](https://github.com/eaudeweb/drupal-sql-dump-action/) in your workflow.

Drush is using the `mysqldump` / `mariadb-dump` (as of with Drush 13) to make queries to the database, therefore we just need to make sure the proper command is used when calling the command. See the example below which is manipulating the `$PATH` variable to achieve this:

```
$> export PATH=/var/www/html/vendor/bin:$PATH
$> which mariadb-dump
/var/www/html/vendor/bin/mariadb-dump
$> ./vendor/bin/drush sql:dump -v --debug --extra-dump="--gdpr-replacements-file=$PWD/anonymize.schema.json" --structure-tables-list=cache,cache_*,watchdog,sessions,history --result-file=$PWD/database.sql
```

## Status and further development

Currently this is a proof of concept to spark a community process.
Especially the `--gdpr-expressions` option is neither handy to write for humans, nor does it scale well.
Here we might need better options.

## Contributors notes

* Note that the project follows [PSR-2](https://www.php-fig.org/psr/psr-2/) for formatting. 
