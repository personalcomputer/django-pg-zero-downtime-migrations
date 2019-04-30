

### Django migrations hacks

Any schema changes can be processed with creation of new table and copy data to it, so just mark unsafe operations that don't have another safe way without downtime as `NO`.

|  # | name                                          | safe | safe alternative              | description |
|---:|-----------------------------------------------|:----:|:-----------------------------:|-------------|
| 19 | `ALTER TABLE ADD CONSTRAINT CHECK`            |      | add as not valid and validate | **unsafe operation**, because you spend time in migration to check constraint
| 20 | `ALTER TABLE DROP CONSTRAINT` (`CHECK`)       | X    |                               | safe operation


\*: main point with migration on production without downtime that your code should correctly work before and after migration, lets look this point closely below

\*\*: postgres will check that all items in column `NOT NULL` that take time, lets look this point closely below

\*\*\*: postgres will have same behaviour when you skip `ALTER TABLE ADD CONSTRAINT UNIQUE USING INDEX` and still unclear difference with `CONCURRENTLY` except difference in locks, lets look this point closely below

\*\*\*\*: lets look this point closely below

\*\*\*\*\*: if you check migration on CI with `python manage.py makemigrations --check` you can't drop column in code without migration creation, so in this case you can be useful *back migration flow*: apply code on all instances and then migrate database

#### Dealing with logic that should work before and after migration

##### New and removing models and columns

Migrations: `CREATE SEQUENCE`, `DROP SEQUENCE`, `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE ADD COLUMN`, `ALTER TABLE DROP COLUMN`.

This migrations are pretty safe, because your logic doesn't work with this data before migration

##### Changes for working logic

Migrations: `ALTER TABLE RENAME TO`, `ALTER TABLE SET TABLESPACE`, `ALTER TABLE RENAME COLUMN`.

For this migration too hard implement logic that will work correctly for all instances, so there are two ways to dealing with it:

1. create new table/column, copy exist data, drop old table/column
2. downtime

##### Create column with default

Migrations: `ALTER TABLE ADD COLUMN SET DEFAULT`.

Standard django's behaviour for creation column with default is populate all values with default. Django don't use database defaults permanently, so when you add new column with default django will create column with default and drop this default at once, eg. new default will come from django code. In this case you can have a gap when migration applied by not all instances has updated and at this moment new rows in table will be without default and probably you need update nullable values after that. So to avoid this case best way is avoid creation column with default and split column creation (with default for new rows) and data population to two migrations (with deployments).

#### Dealing with `NOT NULL` constraint

Postgres check that all column items `NOT NULL` when you applying `NOT NULL` constraint, unfortunately you can't defer this check as for `NOT VALID`. But we have some hacks and alternatives there.

1. Run migrations when load minimal to avoid negative affect of locking.
2. `SET statement_timeout` and try to set `NOT NULL` constraint for small tables.
3. Use `CHECK (column IS NOT NULL)` constraint instead that support `NOT VALID` option with next `VALIDATE CONSTRAINT`, see article for details https://medium.com/doctolib-engineering/adding-a-not-null-constraint-on-pg-faster-with-minimal-locking-38b2c00c4d1c.

#### Dealing with `UNIQUE` constraint

Postgres has two approaches for uniqueness: `CREATE UNIQUE INDEX` and `ALTER TABLE ADD CONSTRAINT UNIQUE` - both use unique index inside. Difference that I see that you cannot apply `DROP INDEX CONCURRENTLY` for constraint. However still unclear what difference for `DROP INDEX` and `DROP INDEX CONCURRENTLY` except difference in locks, but as you see before both marked as safe - you don't spend time in `DROP INDEX`, just wait for lock. So as django use constraint for uniqueness we also have a hacks to use constraint safely.

#### Dealing with `ALTER TABLE ALTER COLUMN TYPE`

Next operations are safe:

1. `varchar(LESS)` to `varchar(MORE)` where LESS < MORE
2. `varchar(ANY)` to `text`
3. `numeric(LESS, SAME)` to `numeric(MORE, SAME)` where LESS < MORE and SAME == SAME

For other operations propose to create new column and copy data to it. Eg. some types can be also safe, but you should check yourself.
