# mysql-to-mongo

A collection of scripts to simplify importing database tables from
[MySQL<sup>*</sup>](http://www.mysql.com/) into [MongoDB](http://www.mongodb.org/).

Using [mongoimport](http://www.mongodb.org/display/DOCS/Import+Export+Tools) to import all
the tables of a relational database into MongoDB can be a bit tedious if you have more than a
handful of tables. This collection of scripts makes the process much less manual.

<sub>* This tool can be used with any database, or even no database.
Only one of the scripts is directly related to MySQL.</sub>


## Features

* Supports comma and tab-delimited data files
* Automatic export SQL generation
* Easily filter exported records by customizing table SELECT SQL
* Easily change exported field values by customizing column SELECT SQL
* Convert data files to UTF-8 before import
* Create collections from joined data files
* Workarounds for known `mongoimport` parsing issues


## Requirements

* [mongoimport](http://www.mongodb.org/display/DOCS/Import+Export+Tools)
* [MySQL](http://www.mysql.com/) (really, all you need is a set of comma or tab-delimited data files)
* [BASH 3.0 or later](http://www.gnu.org/software/bash/)
* [Awk 3.0 or later](http://www.gnu.org/software/gawk/)


## Installation

Download the archive and extract into a folder. Then, to install the package:

	make install

This installs user scripts to `/usr/local/bin` and man pages to `/usr/share/man`.
You can also stage the installation with:

	make DESTDIR=/stage/path install

You can undo the install with:

	make uninstall


## Usage

	my2mo-fields [OPTION]... SCHEMAFILE
	my2mo-export [OPTION]... CSVDIR EXPORTDB
	my2mo-import [OPTION]... CSVDIR IMPORTDB [-- IMPORTOPTIONS]

Run each command with `--help` argument or see the man pages for available OPTIONS.


## Quick Start

Here are the commands necessary to import an entire MySQL database into MongoDB.

	$ mkdir -p my2mo/csvdata
	$ cd my2mo
	$ mysqldump --no-data mydatabase > schema.sql
	$ my2mo-fields schema.sql
	$ my2mo-export csvdata mydatabase
	$ mysql -p -u root < export.sql
	$ my2mo-import csvdata modatabase


## TL;DR

### Step 1: Create table list and fields files

The first information `mongoimport` needs that can take quite some time to produce
from scratch is a list of fields to import for each table. If you want to do
this by hand, have fun. But if you have an SQL file with the database schema,
the `my2mo-fields` script will jumpstart the process.

	my2mo-fields [OPTION]... SCHEMAFILE

`my2mo-fields` reads the database schema SQL generated by a tool such as
[mysqldump](http://dev.mysql.com/doc/refman/5.5/en/mysqldump.html)
or [phpMyAdmin](http://www.phpmyadmin.net/). Each table will have a statement like this:

	CREATE TABLE Table1 (
	  Field1 ...,
	  Field2 ...,
	  ...
	  PRIMARY KEY ...
	) TYPE=...;

`my2mo-fields` creates the file `import.tables` that lists the tables found, and creates a
`TABLE.fields` file for each table listing all the table columns/fields. (The fields files
are grouped together in a `fields` subdirectory.)

#### import.tables

The `import.tables` file lists each table that will be imported.
The first column of the file is the table name.
The optional second column is the SQL to use in the SELECT statement of the export SQL
to filter records. Typically this would be a WHERE or ORDER BY statement.

	# List of tables to import
	# TABLE [SELECT SQL]
	Table1
	Table2 ORDER BY Field1

#### *.fields

Each imported table must have a corresponding fields file that lists the table fields
that will be imported. The first column of the file is the field name.
The optional second column is the SQL to use in the SELECT statement of the export SQL
to select the column data.

	# List of fields to import
	# COLUMN [SELECT SQL]
	Field1
	Field2 IFNULL(Field2, "")

### Step 2: Create SQL commands to create data files

The next step is to export the database tables to comma or tab-delimited data files. This requires
running a series of [mysqldump](http://dev.mysql.com/doc/refman/5.5/en/mysqldump.html)
or [SELECT INTO OUTFILE](http://dev.mysql.com/doc/refman/5.5/en/select.html)
statements against your MySQL database. The `my2mo-export` script facilitates the latter option.
If you already have data files ready for import, you can skip this step.

	my2mo-export [OPTION]... CSVDIR EXPORTDB

`my2mo-export` reads the `import.tables` and `*.fields` files (see previous section) to create
an `export.sql` file containing SQL to execute against your database server.
You can edit the `import.tables` and `*.fields` files before running the command
to comment out tables or fields to exclude or add additional SQL for record selection.

The SQL generated is simply a series of SELECT statements that look like this:

	SELECT ... FROM TABLE INTO OUTFILE "CSVDIR/TABLE.csv" ...;

Redirect this script into a `mysql` command to export each table to a file in `CSVDIR`
(as shown in the Quick Start section).

### Step 3: Import data files into MongoDB

The final step is to run [mongoimport](http://www.mongodb.org/display/DOCS/Import+Export+Tools) for
each table data file you want to import into MongoDB. The script `my2mo-import` makes this easy.

	my2mo-import [OPTION]... CSVDIR IMPORTDB [-- IMPORTOPTIONS]

`my2mo-import` reads the `import.tables` and `*.fields` files (see previous section) and executes
`mongoimport` with the appropriate options. The result is a fresh collection created for each table.
Each command looks like this:

	mongoimport --db IMPORTDB --type csv --drop -c TABLE --file TABLE.csv --fields ... IMPORTOPTIONS

If you need to specify additional options for `mongoimport` (for example, `--host` or `--username`)
just include them at the end of the command line after a `--`.
Details of the import process are saved to the file `mongoimport.log`.

## Data File Joins

`my2mo-import` supports joining two table data files using the `join` command. Joined tables are listed
in a `join.tables` file. This file lists the new table name, the two tables to join,
and the fields to join on. Because `join` requires both data files to be sorted on the join field,
you can specify which field the resulting data should be sorted by before importing
into the database. Use `-` if you don't want to sort the join results. The resulting table
will not include the join field of the righthand table.

	# List of tables to join
	# NewTable Table1.Field Table2.Field FinalSortField
	UserCity User.AddressCityID City.CityID AddressCityID

Note that this join is done on the data files themselves, and not by MySQL. If you want to import
a table that is a result of an SQL join, you can edit the `export.sql` file by hand and create
a corresponding `.fields` file.


## UTF-8 Encoding

If `mongoimport` warns you about invalid characters in your data files, you can run `my2mo-import` with
the `-i` option and it will run `iconv` to create a copy of the data files. The character
set will be changed from ASCII to UTF-8, and invalid characters will be ommited from the data file.
Specify the `-I` option to convert and delete the original data file if no changes are detected.


## Workarounds

Through at least version 1.6.5, `mongoimport` has some serious parsing issues
<sup>[1](http://jira.mongodb.org/browse/SERVER-2379),[2](http://jira.mongodb.org/browse/SERVER-805),[3](http://jira.mongodb.org/browse/SERVER-2604)</sup>
with both comma and tab-delimited data files. Until these issues are resolved,
your best option is to run `my2mo-fields` with the `-W` option and run
`my2mo-export` and `my2mo-import` with the `-t` option. This will create tab-delimited data files
where all tab, carriage return and newline characters (`\t`,`\r`,`\n`) are replaced
with the text `<TAB>`, `<CR>` and `<LF>` respectively
for all `CHAR`, `TEXT` and `BLOB` field types.