---
title: CREATE SCHEMA
summary: The CREATE SCHEMA statement creates a new user-defined schema.
toc: true
---

<span class="version-tag">New in v20.2</span>: The `CREATE SCHEMA` [statement](sql-statements.html) creates a user-defined [schema](sql-name-resolution.html#logical-schemas-and-namespaces) in the current database.

{{site.data.alerts.callout_info}}
You can also create a user-defined schema by converting an existing database to a schema using [`ALTER DATABASE ... CONVERT TO SCHEMA`](convert-to-schema.html).
{{site.data.alerts.end}}

## Required privileges

Only members of the `admin` role can create new schemas. By default, the `root` user belongs to the `admin` role.

## Syntax

~~~
CREATE SCHEMA [IF NOT EXISTS] { <schemaname> | [<schemaname>] AUTHORIZATION {user_name | CURRENT_USER | SESSION_USER} }
~~~

### Parameters

Parameter | Description
----------|------------
`IF NOT EXISTS` | Create a new schema only if a schema of the same name does not already exist within the current database. If one does exist, do not return an error.
`schemaname` | The name of the schema to create, which must be unique within the current database and follow these [identifier rules](keywords-and-identifiers.html#identifiers).
`AUTHORIZATION ...` | Optionally identify a user to be the owner of the schema. You can specify the owner by name, or with the [`CURRENT_USER` or `SESSION_USER` keywords](functions-and-operators.html#special-syntax-forms).<br><br>If a `CREATE SCHEMA` statement has an `AUTHORIZATION` clause, but no `schemaname`, the schema will be named after the specified owner of the schema. If a `CREATE SCHEMA` statement does not have an `AUTHORIZATION` clause, the user executing the statement will be named the owner.

## Example

{% include {{page.version.version}}/sql/movr-statements.md %}

### Create a schema

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA org_one;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name     | owner
---------------------+--------
  crdb_internal      | NULL
  information_schema | NULL
  org_one            | demo
  pg_catalog         | NULL
  pg_extension       | NULL
  public             | admin
(6 rows)
~~~

By default, the user executing the `CREATE SCHEMA` statement is the owner of the schema. For example, suppose you created the schema as user `demo`. `demo` would be the owner of the schema.

### Create a schema if one does not exist

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA org_one;
~~~

~~~
ERROR: schema "org_one" already exists
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA IF NOT EXISTS org_one;
~~~

SQL does not generate an error, even though a new schema wasn't created.

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name     | owner
---------------------+--------
  crdb_internal      | NULL
  information_schema | NULL
  org_one            | demo
  pg_catalog         | NULL
  pg_extension       | NULL
  public             | admin
(6 rows)
~~~

### Create two tables of the same name in different schemas

You can create tables of the same name in the same database if they are in separate schemas.

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA IF NOT EXISTS org_one;
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA IF NOT EXISTS org_two;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name     | owner
---------------------+--------
  crdb_internal      | NULL
  information_schema | NULL
  org_one            | demo
  org_two            | demo
  pg_catalog         | NULL
  pg_extension       | NULL
  public             | admin
(7 rows)
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE org_one.employees (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING,
        desk_no INT UNIQUE
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE org_two.employees (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING,
        desk_no INT UNIQUE
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW TABLES;
~~~

~~~
  schema_name | table_name | type  | owner | estimated_row_count
--------------+------------+-------+-------+----------------------
  org_one     | employees  | table | demo  |                   0
  org_two     | employees  | table | demo  |                   0
(2 rows)
~~~

### Create a schema with authorization

To specify the owner of a schema, add an `AUTHORIZATION` clause to the `CREATE SCHEMA` statement:

{% include copy-clipboard.html %}
~~~ sql
> CREATE USER max WITH PASSWORD 'roach';
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA org_two AUTHORIZATION max;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name     | owner
---------------------+--------
  crdb_internal      | NULL
  information_schema | NULL
  org_two            | max
  pg_catalog         | NULL
  pg_extension       | NULL
  public             | admin
(6 rows)
~~~

If no schema name is specified in a `CREATE SCHEMA` statement with an `AUTHORIZATION` clause, the schema will be named after the user specified:

{% include copy-clipboard.html %}
~~~ sql
> CREATE SCHEMA AUTHORIZATION max;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name     | owner
---------------------+--------
  crdb_internal      | NULL
  information_schema | NULL
  max                | max
  org_two            | max
  pg_catalog         | NULL
  pg_extension       | NULL
  public             | admin
(7 rows)
~~~

When you [use a table without specifying a schema](sql-name-resolution.html#search-path), CockroachDB looks for the table in the `$user` schema (i.e., a schema named after the current user). If no schema exists with the name of the current user, the `public` schema is used.

For example, suppose that you [grant the `demo` role](grant-roles.html) (i.e., the role of the current user `demo`) to the `max` user:

{% include copy-clipboard.html %}
~~~ sql
> GRANT demo TO max;
~~~

Then, `max` [accesses the cluster](cockroach-sql.html) and creates two tables of the same name, in the same database, one in the `max` schema, and one in the `public` schema:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql --url 'postgres://max:roach@host:port/db?sslmode=require'
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE max.accounts (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING,
        balance DECIMAL
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE public.accounts (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING,
        balance DECIMAL
);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SHOW TABLES;
~~~

~~~
  schema_name | table_name | type  | owner | estimated_row_count
--------------+------------+-------+-------+----------------------
  max         | accounts   | table | max   |                   0
  public      | accounts   | table | max   |                   0
(2 rows)
~~~

`max` then inserts some values into the `accounts` table, without specifying a schema:

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO accounts (name, balance) VALUES ('checking', 1000), ('savings', 15000);
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM accounts;
~~~

~~~
                   id                  |   name   | balance
---------------------------------------+----------+----------
  7610607e-4928-44fb-9f4e-7ae6d6520666 | savings  |   15000
  860b7891-cde4-4aff-a318-f928d47374bc | checking |    1000
(2 rows)
~~~

Because `max` is the current user, all unqualified `accounts` table names resolve as `max.accounts`, and not `public.accounts`.

{% include copy-clipboard.html %}
~~~ sql
> SELECT * FROM public.accounts;
~~~

~~~
  id | name | balance
-----+------+----------
(0 rows)
~~~

## See also

- [`SHOW SCHEMAS`](show-schemas.html)
- [`SET SCHEMA`](set-schema.html)
- [`DROP SCHEMA`](drop-schema.html)
- [`ALTER SCHEMA`](alter-schema.html)
- [Other SQL Statements](sql-statements.html)
- [Online Schema Changes](online-schema-changes.html)