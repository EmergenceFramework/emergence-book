# Migrations

Migrations are scripts designed to run once on each instance they're deployed to in the order they were created. They are most useful for transforming database structure but can also be used to check or update configuration or other application customizations.

## Migrations state storage

The status of each migration script is tracked in the `_e_migrations` table. New scripts have no record in the table and are considered pending and in need of execution. Once started, a migration script cannot be run again without manually resetting its status in the `_e_migrations` table.

## Developer user interface

A user interface is available for Developer+ users online at `/site-admin/migrations`. From here you can view a list of all **pending**, **started**, **skipped**, **failed**, and **executed** migrations and execute any that are pending.

## Philosophy

Migration scripts in emergence don't have the usually `up`/`down` workflow that migration frameworks usually provide. Instead, it is expected that a website/database be snapshotted before migrations are run as this provides a more reliable recovery process in the case of failed migrations. Each script implements only one routine: upgrade if needed. The goal of every migration script is to first check if it can do nothing and return the status **skipped**, and then execute the migration and return the status **executed** or **failed**. A migration should be safe to run multiple times and return **skipped** when it can test that it has been run already.

Migrations should make incremental changes, only changing that which it checked for, such as adding a database column or changing its type. Table schemas can be generated in a number of ways and can be augmented by configuration from other packages, so moving a table to a total known state isn't the goal.

## Building a migration

### Creating script file

Create a new file under the `php-migrations` tree, placing it under a folder reflecting the highest common PHP namespace of the application/database elements it affects. Each file starts with a datestamp in the format `YYYYMMDD` which is used to order the execution of migrations. The datestame can optionally further include a time component in the format `YYYYMMDDHHMMSS` to order the execution of migrations added on the same day. Many older migrations filled the time component with `000000` but this can be ommitted now with the same effect.

### Script scope

Each migration is executed in a closed scope with three variables predefined:

* `$migration`: An array containing the stored metadata about the transaction
* `$migrationNode`: A `SiteFile` instance for the current script being executed
* `$resetMigrationStatus`: A `callable` you can execute to delete the stored status for the current migration \(useful while debugging, see below\)

Additionally, the migration runs in the scope of a member of the `Emergence\SiteAdmin\MigrationsRequestHandler` class and has access to a number of protected static methods it provides:

* `static::tableExists($tableName)`
* `static::columnExists($tableName, $columnName)`
* `static::getColumns($tableName)`
* `static::getColumnNames($tableName)`
* `static::getColumn($tableName, $columnName)`
* `static::getColumnType($tableName, $columnName)`
* `static::getColumnKey($tableName, $columnName)`
* `static::getColumnDefault($tableName, $columnName)`
* `static::getColumnIsNullable($tableName, $columnName)`
* `static::getConstraints($tableName)`
* `static::getConstraint($tableName, $constraintName)`
* `static::addColumn($tableName, $columnName, $definition, $position = null)`
* `static::addIndex($tableName, $indexName, array $columns = [], $type = null)`
* `static::dropColumn($tableName, $columnName)`

All output is captured and reported on after a migration is executed but not \(currently\) saved.

### Debugging

The best workflow for debugging a migration is to dump and reload all application tables \(including `_e_migrations` but excluding `_e_file*` VFS tables if present\) between each execution.

During the development though, you might find it helpful to call `$resetMigrationStatus()` at the beggining of your script or `return static::STATUS_DEBUG` to erase the `_e_migrations` record that would prevent you from running it over and over again.

## Example migrations

### Add a column

This migration from `slate-cbl` is about as simple as it gets:

```php
<?php

namespace Slate\CBL\Tasks;


// skip if Task table does not exist or already has ClonedTaskID
if (!static::tableExists(Task::$tableName)) {
    printf("Skipping migration because table `%s` does not yet exist\n", Task::$tableName);
    return static::STATUS_SKIPPED;
}

if (static::columnExists(Task::$tableName, 'ClonedTaskID')) {
    printf("Skipping migration because column `%s`.`ClonedTaskID` already exists\n", Task::$tableName);
    return static::STATUS_SKIPPED;
}


// add ClonedTaskID column to Task table
static::addColumn(Task::$tableName, 'ClonedTaskID', 'int unsigned NULL default NULL', 'AFTER `ParentTaskID`');


// finish
return static::STATUS_EXECUTED;
```

### Move column to parent record

This migration, also from `slate-cbl`, is about as complex as it gets:

```php
<?php

namespace Slate\CBL\Tasks;

use DB;
use HandleBehavior;


// skip if Task table does not exist or already has SectionID
if (!static::tableExists(Task::$tableName)) {
    printf("Skipping migration because table `%s` does not yet exist\n", Task::$tableName);
    return static::STATUS_SKIPPED;
}

if (static::columnExists(Task::$tableName, 'SectionID')) {
    printf("Skipping migration because column `%s`.`SectionID` already exists\n", Task::$tableName);
    return static::STATUS_SKIPPED;
}


// find existing SectionIDs for all associated StudentTask records, clone Task records as needed
$taskColumnNames = array_diff(static::getColumnNames(Task::$tableName), ['ID', 'Handle']);
$taskSectionIds = DB::arrayTable('TaskID', 'SELECT DISTINCT TaskID, SectionID FROM `%s`', StudentTask::$tableName);
$taskSectionId = [];

foreach ($taskSectionIds as $taskId => $studentTaskSections) {
    // original task gets first section
    if ($studentTaskSection = array_shift($studentTaskSections)) {
        $taskSectionId[$taskId] = $studentTaskSection['SectionID'];
    }

    // no cloning is needed
    if (count($studentTaskSections) == 0) {
        continue;
    }

    $taskTitle = DB::oneValue('SELECT Title FROM `%s` WHERE ID = %u', [Task::$tableName, $taskId]);

    // clone for any/each additional section
    while ($studentTaskSection = array_shift($studentTaskSections)) {
        $cloneHandle = HandleBehavior::getUniqueHandle(Task::class, $taskTitle);
        DB::nonQuery(
            'INSERT INTO `%1$s` (`ID`, `Handle`, `%4$s`) SELECT NULL, "%3$s", `%4$s` FROM `%1$s` WHERE ID = %2$u',
            [
                Task::$tableName,
                $taskId,
                DB::escape($cloneHandle),
                implode('`, `', $taskColumnNames)
            ]
        );

        $taskSectionId[DB::insertID()] = $studentTaskSection['SectionID'];
        printf("Cloning task %u for section %u\n", $taskId, $studentTaskSection['SectionID']);
    }
}


// add SectionID column to Task table
static::addColumn(Task::$tableName, 'SectionID', 'int unsigned NULL default NULL', 'AFTER `ModifierID`');
static::addIndex(Task::$tableName, 'SectionID');


// clone Task for each SectionID where multiple are associated
printf("Setting SectionID for %u Task records\n", count($taskSectionId));
foreach ($taskSectionId as $taskId => $sectionId) {
    DB::nonQuery('UPDATE `%s` SET SectionID = %u WHERE ID = %u', [
        Task::$tableName,
        $sectionId,
        $taskId
    ]);
}


// remove SectionID column from StudentTask table
static::dropColumn(StudentTask::$tableName, 'SectionID');


// finish
return static::STATUS_EXECUTED;
```
