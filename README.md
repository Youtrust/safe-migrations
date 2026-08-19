# 🦺 safe-migrations

Make your migrations safe 

## 🪄 Features

- PG 13+
- PHP 8.3
- Doctrine Migration

## 🤷 Why?

Because SQL migrations can execute heavy queries on database which can slow down your application.

## ⬆️ Upgrading from `yousign/safe-migrations`

This package was published as `yousign/safe-migrations` up to v1.0.7, under the `Yousign\SafeMigrations\`
namespace. Following the company rename to Youtrust, v2.0.0 moves to `youtrust/safe-migrations` and the
`Youtrust\SafeMigrations\` namespace. There is no functional change between v1.0.7 and v2.0.0.

```shell
$ composer remove yousign/safe-migrations
$ composer req youtrust/safe-migrations:^2.0
```

Then replace every `Yousign\SafeMigrations\` prefix with `Youtrust\SafeMigrations\`. Two spots are easy to
miss because they live in configuration rather than in PHP code:

- the middleware service id in `services.yaml`
- the `use` statement in your `migration.php.tpl` template

`yousign/safe-migrations` is abandoned and no longer maintained. It will receive no further
release of any kind, security fixes included. Upgrade to `youtrust/safe-migrations`.

## ⚙️ Config

_*For Symfony 6.4, 7.x or 8.x, with Doctrine DBAL 3 or 4_

Install in your project 

```shell
$ composer req youtrust/safe-migrations
```

Declare the Middleware in your `services.yaml`

```yaml
parameters:
  env(ENABLE_RETRY_LOCK_TIMEOUT): false
  
services:
  Youtrust\SafeMigrations\Doctrine\DBAL\Driver\Middleware\RetryLockTimeoutMiddleware:
    $isEnabled: '%env(bool:ENABLE_RETRY_LOCK_TIMEOUT)%'
```

Create a migration template `migration.php.tpl`

```php
<?php

declare(strict_types=1);

namespace <namespace>;

use Doctrine\DBAL\Schema\Schema;
use Youtrust\SafeMigrations\Doctrine\Migration;

class <className> extends Migration
{
    public function up(Schema $schema): void
    {
<up>
    }
}
```

Set this template as default for your migrations in `doctrine_migrations.yaml`

```yaml
doctrine_migrations:
  custom_template: "%kernel.project_dir%/migrations/migration.php.tpl"
```

Enable the retry on lock through the env var in you `.env<.environment>`

```dotenv
ENABLE_RETRY_LOCK_TIMEOUT=true
```

That's it ☕

## ▶️ Usage

### Migration class

When you generate a new migration, this one extend the `Migration` class of the library which expose the following safe methods.

Each of these methods will generate the right set of SQL requests to make the requested query safe.

<details><summary>Create table</summary>

```php
$this->createTable(table: 'test', columnDefinitions: [
    'id UUID NOT NULL', 
    'PRIMARY KEY(id)',
])
```
</details>

<details><summary>Add foreign key</summary>

```php
$this->addForeignKey(table: 'address', name: 'fk_address_contact', column: 'contact', referenceTable: 'contact', referenceColumn: 'id', options: 'ON DELETE CASCADE NOT DEFERRABLE INITIALLY IMMEDIATE')
```
</details>

<details><summary>Rename constraint</summary>

```php
$this->renameConstraint(table: 'address', from: 'id_pkey', to: 'pkey_id')
```
</details>

<details><summary>Comment on column</summary>

```php
$this->commentOnColumn(table: 'address', name: 'name', comment: null)
```
</details>

<details><summary>Add index</summary>

> Note: Adding an index on a table will execute an "analyze" on all columns of the table to update statistics

```php
$this->addIndex(name: 'idx_contact_email', table: 'contact', columns: ['email'], unique: false, usingMethod: 'GIN', where: 'country = "France"')
```
</details>

<details><summary>Drop index</summary>

```php
$this->dropIndex(name: 'idx_contact_email')
```
</details>

<details><summary>Rename index</summary>

```php
$this->renameIndex(from: 'idx_email_signer', to: 'idx_signer_email')
```
</details>

<details><summary>Add column</summary>

```php
$this->addColumn(table: 'contact', name: 'mobile', type: 'text', defaultValue: null, nullable: true)
```
</details>

<details><summary>Drop column</summary>

```php
$this->dropColumn(table: 'contact', name: 'landline')
```
</details>

<details><summary>Set default on column</summary>

```php
$this->setDefaultOnColumn(table: 'contact', name: 'email', value: "'noreply@undefined.org'")
```
</details>

<details><summary>Drop default on column</summary>

```php
$this->dropDefaultOnColumn(table: 'contact', name: 'email')
```
</details>

<details><summary>Set column nullable</summary>

```php
$this->setColumnNullable(table: 'contact', name: 'email')
```
</details>

<details><summary>Set column not nullable</summary>

```php
$this->setColumnNotNullable(table: 'contact', name: 'email')
```
</details>

### Migration execution

If there is a lock while executing a Doctrine migration, the migration will throw a `DriverException` (Doctrine\DBAL\Driver\Exception).

If the SQLSTATE value is **55P03** (_lock_not_available_), the query will be retried up to 3 times with a 10s interval before throwing the thrown exception if it does not succeed.

You will get the following output:

<details><summary>Failed after the 3 retry</summary>

```shell
$ bin/symfony console d:m:m

[notice] Migrating up to DoctrineMigrations\Version20231224200000
09:30:38 WARNING   [app] (1/3) Lock timeout reached: retrying in 10 seconds... ["sql" => "ALTER TABLE test_retry ADD COLUMN name text DEFAULT NULL"]
09:30:51 WARNING   [app] (2/3) Lock timeout reached: retrying in 10 seconds... ["sql" => "ALTER TABLE test_retry ADD COLUMN name text DEFAULT NULL"]
09:31:04 WARNING   [app] (3/3) Lock timeout reached: retrying in 10 seconds... ["sql" => "ALTER TABLE test_retry ADD COLUMN name text DEFAULT NULL"]
[error] Migration DoctrineMigrations\Version20231224200000 failed during Execution. Error: "An exception occurred while executing a query: SQLSTATE[55P03]: Lock not available: 7 ERROR:  canceling statement due to lock timeout"
09:31:17 CRITICAL  [console] Error thrown while running command "'d:m:m'". Message: "An exception occurred while executing a query: SQLSTATE[55P03]: Lock not available: 7 ERROR:  canceling statement due to lock timeout" - An exception occurred while executing a query: SQLSTATE[55P03]: Lock not available: 7 ERROR:  canceling statement due to lock timeout ["exception" => Doctrine\DBAL\Exception\DriverException^ { …},"command" => "'d:m:m'","message" => "An exception occurred while executing a query: SQLSTATE[55P03]: Lock not available: 7 ERROR:  canceling statement due to lock timeout"]
```
</details>

<details><summary>Succeed after 1 retry</summary>

```shell
bin/symfony console d:m:m

[notice] Migrating up to DoctrineMigrations\Version20231224200000
09:28:54 WARNING   [app] (1/3) Lock timeout reached: retrying in 10 seconds... ["sql" => "ALTER TABLE test_retry ADD COLUMN name text DEFAULT NULL"]
[notice] finished in 15446.1ms, used 38.5M memory, 2 migrations executed, 13 sql queries

[OK] Successfully migrated to version: DoctrineMigrations\Version20231224200000
```
</details>

## 📋 FAQ

<details><summary>Does it work with Migration bundle ?</summary>

Yes, of course. There is no incompatibility between this library and the [doctrine/doctrine-migrations-bundle](https://symfony.com/bundles/DoctrineMigrationsBundle/current/index.html).
</details>

## 🔗 References

- [PostgreSQL doc: Multiversion Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL doc: lock behavior](https://www.postgresql.org/docs/current/explicit-locking.html): must read, it will help you understand a lot of things about PG locks.
 
  *For example, some DDL queries require an exclusive lock on a table. When the table
  is concurrently accessed and modified by other processes, acquiring the lock may take
  a while. The lock request is waiting in a queue, and it may also block other queries
  on this table once it has been enqueued.*

- Examples of other safe migration frameworks:
    - [Doctolib Framework for safe migrations](https://github.com/doctolib/safe-pg-migrations)
    - [Braintree Framework for safe migrations](https://github.com/braintree/pg_ha_migrations)
    - https://squawkhq.com/
    - https://github.com/ankane/strong_migrations
  
- Great article to introduce to ZDD

  [Les Patterns des Géants du Web – Zero Downtime Deployment - OCTO Talks !](https://blog.octo.com/zero-downtime-deployment/)

- Very good resources to know how to schema changes while keeping backward compatibility

  https://databaserefactoring.com

- GitLab explains to contributors how they manage ZDD with their database
  
  https://docs.gitlab.com/ee/development/what_requires_downtime.html

- Article that explain best practices for zero downtime schema changes and many tips
  
  [PostgreSQL at Scale: Database Schema Changes Without Downtime](https://medium.com/paypal-tech/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680)

- Very good video that explains a lot of things about ZDD with Database schema changes:

  [Montée de version sans interruption (Nelson Dionisi)](https://www.youtube.com/watch?v=pIkA-aPtkNs&list=PLTbQvx84FrAQwUMLVvcZu4DZwS1qGxKyM&index=16)

- Another good video on the subject but in English this time:
  
  [PHP UK Conference 2018 - Michiel Rook - Database Schema Migrations with Zero Downtime](https://www.youtube.com/watch?v=un-vdrVAX-A)

- Doctolib explains how they manage their migration without downtime, we basically implemented the same thing.

## 🤝 Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

After writing your fix/feature, you can run following commands to make sure that everything is still ok.

```bash
# Install dev dependencies
$ make vendor

# Running tests and quality tools locally
$ make tests
```

## Authors

- [Benoit GALATI](https://github.com/B-Galati)
- [Nicolas DOUSSON](https://github.com/ndousson)