# Lesson 8: Models, RecordManagers and Migrations, part 1

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

## Introduction

In this lesson, you will revisit the most important backend building blocks of Hubleto: `Model`, `RecordManager`, and `Migration`. Lessons 3 through 7 already used these concepts in practice. Here, we will look at them in a more structured way.

The examples in this lesson come mainly from the Contacts app, Workflow app, and Worksheets app.

The main questions are:

* what belongs in the model,
* what belongs in the record manager,
* how relations are declared,
* how records are created, updated, and deleted,
* and how migrations translate model structure into SQL tables and foreign keys.

> **What you will learn:**
>
> * What the responsibilities of `Model` and `RecordManager` are in Hubleto.
> * How relations are defined in the model and mirrored in the record manager.
> * How `belongsTo` and `hasMany` are used in real Hubleto apps.
> * Which built-in record APIs Hubleto uses for create, update, save, and delete.
> * How `prepareReadQuery()` is customized in real applications.
> * How migrations are organized and why tables and foreign keys are installed separately.

## 1. Model vs. RecordManager

In Hubleto, the **description of a model** is separated from the **engine that reads and writes records**.

The model class usually defines:

* SQL table name,
* record manager class,
* relation metadata,
* column definitions,
* DescriptionAPI configuration,
* callbacks such as `onBeforeCreate()` or `onAfterUpdate()`.

The record manager usually defines:

* the actual Eloquent relations such as `belongsTo()` and `hasMany()`,
* query customization in methods such as `prepareReadQuery()`,
* optional lookup and sorting customization,
* lower-level data loading behavior.

In practice, this means:

* the model stores metadata such as `$table`, `$recordManagerClass`, and `$relations`,
* the default Eloquent record manager handles query preparation, relation loading, and record save/delete logic,
* the ERP record manager adds standard visibility filtering such as owner, manager, team, and sharing rules.

In other words:

* **Model** = what the business object is.
* **RecordManager** = how records of that business object are loaded and manipulated.

## 2. Anatomy of a real model

A good simple example is the `Contact` model from the Contacts app.

Its core properties look like this:

```php
class Contact extends \Hubleto\Erp\Model
{
  public string $table = 'contacts';
  public string $recordManagerClass = RecordManagers\Contact::class;

  public array $relations = [
    'CUSTOMER' => [ self::BELONGS_TO, Customer::class, 'id_customer' ],
    'VALUES' => [ self::HAS_MANY, Value::class, 'id_contact', 'id' ],
    'TAGS' => [ self::HAS_MANY, ContactTag::class, 'id_contact', 'id' ],
  ];
}
```

This tells Hubleto three important things:

1.  Records are stored in the `contacts` table.
2.  Database work is handled by `RecordManagers\Contact`.
3.  The contact has one parent customer and multiple child records such as values and tags.

The same model also defines its columns in `describeColumns()`. For example, `first_name`, `last_name`, `id_customer`, `virt_email`, and `virt_tags` are all described there. This is why the model is not only a database definition. It is also the central description of how the record should be understood and presented.

## 3. Relations are defined in two places

This is one of the most important Level 1 rules.

When you use the standard Eloquent-based record manager, a relation is declared:

1.  in the model's `$relations` array, and
2.  again in the record manager as an Eloquent relation method.

### Why the model needs `$relations`

The `$relations` array gives Hubleto structured metadata that is later used by features such as:

* relation-aware reading,
* nested save and validation,
* form and table data preparation,
* recursive relation loading with controlled depth.

The basic structure is:

```php
'RELATION_NAME' => [ relationType, relatedModelClass, foreignKey, localKey ]
```

A typical relation declaration looks like this:

```php
public array $relations = [
  'CUSTOMER' => [ self::BELONGS_TO, Customer::class, 'id_customer' ],
  'VALUES' => [ self::HAS_MANY, Value::class, 'id_contact', 'id' ],
  'TAGS' => [ self::HAS_MANY, ContactTag::class, 'id_contact', 'id' ],
];
```

### Why the record manager needs relation methods

The record manager still needs actual Eloquent relations so that calls such as `with('VALUES')`, `with('TAGS')`, or `with('STEPS')` work correctly.

Example from the Contacts app:

```php
public function CUSTOMER(): BelongsTo
{
  return $this->belongsTo(Customer::class, 'id_customer');
}

public function VALUES(): HasMany
{
  return $this->hasMany(Value::class, 'id_contact', 'id');
}

public function TAGS(): HasMany
{
  return $this->hasMany(ContactTag::class, 'id_contact', 'id');
}
```

Notice an important detail:

* in the **model**, the relation points to the **model class**,
* in the **record manager**, the Eloquent relation points to the **record manager class**.

This duplication is intentional. The model describes the relation, and the record manager implements it.

## 4. Practical relation examples

### `belongsTo`: Contact -> Customer

This is the standard "many contacts belong to one customer" pattern.

Model definition:

```php
'CUSTOMER' => [ self::BELONGS_TO, Customer::class, 'id_customer' ]
```

Record manager definition:

```php
public function CUSTOMER(): BelongsTo
{
  return $this->belongsTo(Customer::class, 'id_customer');
}
```

Meaning:

* the `contacts` table stores `id_customer`,
* each contact points to one customer,
* Hubleto can load the related customer as `CUSTOMER`.

### `hasMany`: Workflow -> WorkflowStep

This is the standard parent-child pattern.

From the Workflow app:

```php
public array $relations = [
  'STEPS' => [ self::HAS_MANY, WorkflowStep::class, 'id_workflow', 'id' ]
];
```

And in the Workflow record manager:

```php
public function STEPS(): HasMany
{
  return $this->hasMany(WorkflowStep::class, 'id_workflow', 'id')
    ->orderBy('order', 'asc');
}
```

Meaning:

* one workflow has many workflow steps,
* the child table stores `id_workflow`,
* steps are returned in business order, not just insertion order.

### `hasMany`: Contact -> Value

This is another useful real-world example because the child records are not only decorative. They are part of the contact's real data model.

```php
'VALUES' => [ self::HAS_MANY, Value::class, 'id_contact', 'id' ]
```

and:

```php
public function VALUES(): HasMany
{
  return $this->hasMany(Value::class, 'id_contact', 'id');
}
```

This pattern is used when one master record owns a list of subrecords.

## 5. Built-in record manipulation API

Hubleto does not rely only on raw Eloquent calls. It has its own record-oriented API layer on top of Eloquent.

Hubleto provides standard record API endpoints such as:

* `api/record/get`
* `api/record/load-table-data`
* `api/record/lookup`
* `api/record/save`
* `api/record/delete`

These are part of the standard records API.

### `recordSave()` is the main save entry point

In the standard save flow, the backend calls:

```php
$savedRecord = $this->model->record->recordSave(
  $record,
  0,
  $saveRelations
);
```

This is the important part. Standard table and form saving does not directly call `create()` or `update()`. It goes through Hubleto's `recordSave()` API.

The `recordSave()` method handles:

* create vs. update decision,
* permission checks,
* validation,
* callbacks,
* normalization,
* saving selected child relations.

That is why `recordSave()` is more than a thin wrapper around Eloquent.

### `recordCreate()` and `recordDelete()`

The same record manager also provides:

* `recordCreate(array $record): array`
* `recordDelete(int|string $id): int`
* `recordUpdate(array $record, array $originalRecord = []): array`

These are useful in internal application code.

For example, the Workflow app seeds default workflow data during installation using:

```php
$idWorkflow = $mWorkflow->record->recordCreate([
  "name" => $this->translate('Deals'),
  "show_in_kanban" => 1,
  "order" => 3,
  "group" => "deals",
])['id'];

$mWorkflowStep->record->recordCreate([
  'name' => $this->translate('Prospecting'),
  'order' => 1,
  'color' => '#838383',
  'id_workflow' => $idWorkflow,
  'tag' => 'deal-prospecting',
]);
```

This is a good example of when `recordCreate()` is appropriate:

* the app is inserting known bootstrap data,
* it already knows the values to store,
* it is not saving a nested UI form payload.

Unlike `recordSave()`, `recordCreate()` is a more direct helper. It is useful for controlled inserts, but it is not the full relation-aware form-save pipeline.

For delete operations, the standard delete flow calls `recordDelete($id)`, and `recordDelete()` performs permission checks before removing the row.

### Practical rule for Level 1

Use this mental model:

* `recordSave()` = the standard Hubleto save pipeline for UI and relation-aware saves.
* `recordCreate()` = useful for controlled inserts such as app installation, seed data, or focused internal code.
* raw Eloquent methods such as `create()`, `update()`, `delete()` still exist, but they bypass some Hubleto-specific lifecycle logic.

## 6. Customizing `prepareReadQuery()`

`prepareReadQuery()` is one of the most important customization points in a record manager.

At the beginning, `prepareReadQuery()` prepares:

* base selects,
* joins,
* included relations.

Then the ERP layer adds standard restrictions such as:

* owner filtering,
* manager filtering,
* team filtering,
* shared-with filtering,
* junction-table filtering from URL parameters.

Your app-level record manager then extends that behavior with business-specific filtering.

### Example 1: Contacts filtered by customer

From the Contacts app:

```php
public function prepareReadQuery(mixed $query = null, int $level = 0, array|null $includeRelations = null): mixed
{
  $query = parent::prepareReadQuery($query, $level, $includeRelations);
  $query = $query->orderBy('is_primary', 'desc');

  $hubleto = \Hubleto\Erp\Loader::getGlobalApp();
  if ($hubleto->router()->urlParamAsInteger("idCustomer") > 0) {
    $query = $query->where($this->table . '.id_customer', $hubleto->router()->urlParamAsInteger("idCustomer"));
  }

  return $query;
}
```

This is a very common pattern:

1.  call `parent::prepareReadQuery()` first,
2.  read URL parameters,
3.  add business filters,
4.  return the modified query.

### Example 2: Worksheets filtered by task, project, and period

The Worksheets app shows a more advanced version.

It filters by:

* `idTask`,
* `idProject`,
* selected period,
* selected workers,
* grouping filters.

One particularly useful part is this:

```php
if ($idProject > 0) {
  $mProjectTask = $hubleto->getService(ProjectTask::class);

  $projectTasksIds = $mProjectTask->record->prepareReadQuery()
    ->where($mProjectTask->table . '.id_project', $idProject)
    ->pluck('id_task')
    ?->toArray();

  if (count($projectTasksIds) == 0) $projectTasksIds = [0];

  $query = $query->whereIn($this->table . '.id_task', $projectTasksIds);
}
```

This shows that `prepareReadQuery()` is not only for one extra `where()`. It is also the place for real business reading rules.

### Good practices for `prepareReadQuery()`

* Always start with `parent::prepareReadQuery(...)` unless you have a very special reason not to.
* Keep the method focused on read behavior.
* Use URL parameters and filters deliberately.
* Avoid placing unrelated write logic here.
* If the method becomes too large, extract helper methods.

## 7. Migrations: how models become SQL tables

In Hubleto, migrations are discovered from the model's `Migrations` folder and executed through the model's own upgrade methods.

The model loads migration classes from:

* the same namespace as the model,
* the `Models/Migrations` subfolder,
* files whose names start with the model's short name.

The current migration method names are:

* `upgradeSchema()`
* `downgradeSchema()`
* `upgradeForeignKeys()`
* `downgradeForeignKeys()`

You may sometimes see other method names in older examples. In the examples used here, the method names above are the ones in use.

### Why tables and foreign keys are separated

The upgrade process runs pending table migrations first.

Then it runs pending foreign-key migrations afterward.

This separation prevents dependency-order problems. If model A references model B, both tables must exist before the foreign key is added.

### Practical example: Workflow and WorkflowStep

From the Workflow app migration:

```php
public function upgradeSchema(): void
{
  $this->db->execute("SET foreign_key_checks = 0;
create table `workflows` (
 `id` int(8) primary key auto_increment,
 `name` varchar(255),
 `order` int(255),
 `description` varchar(255),
 `show_in_kanban` int(1),
 `group` varchar(255),
 index `id` (`id`),
 index `order` (`order`),
 index `show_in_kanban` (`show_in_kanban`),
 INDEX `group` (`group`)) ENGINE = InnoDB;
SET foreign_key_checks = 1;");
}
```

And from the migration for workflow steps:

```php
public function upgradeSchema(): void
{
  $this->db->execute("SET foreign_key_checks = 0;
create table `workflow_steps` (
 `id` int(8) primary key auto_increment,
 `id_workflow` int(8) NULL default NULL,
 `name` varchar(255),
 `order` int(255),
 `color` char(10),
 `tag` varchar(255),
 `probability` int(255),
 index `id` (`id`),
 index `id_workflow` (`id_workflow`),
 index `order` (`order`),
 index `probability` (`probability`)) ENGINE = InnoDB;
SET foreign_key_checks = 1;");
}

public function upgradeForeignKeys(): void
{
  $this->db->execute("ALTER TABLE `workflow_steps`
    ADD CONSTRAINT `fk_fabc826acfa6488d209bc1f03431b9ef`
    FOREIGN KEY (`id_workflow`)
    REFERENCES `workflows` (`id`)
    ON DELETE RESTRICT
    ON UPDATE RESTRICT;");
}
```

This is the direct SQL counterpart of the relation:

```php
'STEPS' => [ self::HAS_MANY, WorkflowStep::class, 'id_workflow', 'id' ]
```

The model relation explains the PHP-level meaning. The migration makes it physically true in the database.

## 8. What to study next

If you want to turn this lesson into hands-on practice, review these parts in this order:

1.  The base `Model` class.
2.  The default Eloquent record manager.
3.  The ERP record manager extension.
4.  The `Contact` model and its record manager in the Contacts app.
5.  The `Workflow` model and its record manager in the Workflow app.
6.  The workflow migrations in the Workflow app.
7.  The `Activity` record manager in the Worksheets app.

When you read them, answer these questions:

* Which parts are metadata and which parts are executable query logic?
* Which relations are declared in the model?
* Which relations are implemented as Eloquent methods?
* Which read rules come from the base layer, which from the ERP layer, and which from the app?
* Which migration creates the table and which migration adds the foreign key?

## 9. Common mistakes

Developers usually make these mistakes at this stage:

### Mistake 1: Defining a relation only in the model

If you define a relation in `$relations` but forget the matching Eloquent relation method in the record manager, relation loading will not behave correctly.

### Mistake 2: Putting read filtering into controllers instead of `prepareReadQuery()`

Controllers should stay thin. Query restrictions belong in the record manager.

### Mistake 3: Using raw Eloquent writes everywhere

Raw `create()` and `update()` are valid tools, but standard Hubleto save flows are built around `recordSave()`, `recordCreate()`, and `recordDelete()`.

### Mistake 4: Treating migrations as optional

If the model changed but the migration did not, your code and database schema are no longer describing the same system.

### Mistake 5: Forgetting that ERP record managers already filter visibility

When you extend the ERP record manager, some read restrictions are already included. Do not accidentally duplicate them in every app.

## Best practices

* Keep the model as the authoritative description of the business object.
* Keep the record manager as the authoritative place for relation methods and read-query customization.
* Use relation names consistently between `$relations` and record manager methods.
* Call `parent::prepareReadQuery()` first.
* Prefer `recordSave()` for normal form-driven save flows.
* Use migrations to keep schema evolution explicit and repeatable.

## Study material

| Resource | Description |
| --- | --- |
| [Lesson 3](lesson-3) | First introduction to backend app development. |
| [Lesson 4](lesson-4) | How controllers, views, and React tables/forms consume backend data. |
| [Lesson 7](lesson-7) | How ReactUi tables and forms depend on model descriptions and record APIs. |
| [Models](../../docs/framework/models) | Overview of model structure and responsibilities. |
| [Record manager](../../docs/framework/models/record-manager) | Explanation of record managers and Eloquent-backed data access. |
| [Records API](../../docs/framework/models/records-api) | Framework CRUD endpoints and save/delete flow. |
| [Migrations](../../docs/framework/models/migrations) | Overview of migration-based schema versioning. |
| [Description API](../../docs/erp/advanced-development/description-api) | UI description layer that sits on top of models. |

## Videos

The webinar recording for this lesson will be added after the live session.

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification course](../../certification).
