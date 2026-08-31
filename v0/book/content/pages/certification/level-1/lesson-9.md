# Lesson 9: Models, RecordManagers and Migrations, part 2

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

## Introduction

[Lesson 8](lesson-8) introduced models, record managers, relations, the record
API, query preparation, and initial migrations.

This lesson focuses on using those features in larger forms and tables. The
goal is not to reproduce framework internals. It is to understand which
standard extension point to use for each requirement.

> **What you will learn:**
>
> - How to load only the relations required by a table or form.
> - Which RecordManager method should handle each read customization.
> - How to configure useful lookup labels and lookup restrictions.
> - How to save parent and child records through `recordSave()`.
> - When validation and normalization are applied.
> - How record permissions are enforced during save, read, and delete operations.
> - How to define indexes and evolve an installed database schema.

## 1. Model, RecordManager, and migration responsibilities

Keep these responsibilities separate:

| Layer | Put this here |
| --- | --- |
| Model | Columns, relation metadata, indexes, lookup labels, relation-loading settings, record permissions |
| RecordManager | Eloquent relation methods and executable query behavior |
| Migration | SQL tables, columns, indexes, and foreign keys |

For one relation, this normally means:

1. describe it in the model's `$relations` array,
2. implement the matching Eloquent method in the record manager,
3. create the required column, index, and foreign key in a migration,
4. explicitly load or save the relation only where it is needed.

Changing model metadata does not change an existing database. Likewise,
creating a database foreign key does not define the relation Hubleto uses when
loading records.

## 2. Loading relations for tables and forms

Use `getRelationsIncludedInLoadTableData()` and
`getRelationsIncludedInLoadFormData()` when a table and form need different
related data.

For example, a form that edits child lines can load only that relation:

```php
public function getRelationsIncludedInLoadFormData(): array|null
{
  return ['LINES'];
}
```

Use relation names exactly as declared in `$relations` and implemented by the
record manager.

In normal form development, the default direct-relation depth is usually
enough. Tables default to depth `0`, so a table that explicitly needs a direct
relation must also override `getMaxReadLevelForLoadTableData()` to return `1`.
Only change the form depth when the form genuinely consumes relations below
the direct relation level.

The practical rule is simple: load only relations the screen uses, and avoid
returning child collections in list responses unless the table needs them.

## 3. Choosing a read-query extension point

The standard table loader already supports:

- relation loading,
- full-text search,
- column search,
- ordering,
- pagination,
- record permissions.

The form loader prepares the normal read query, restricts it to one record, and
loads the relations configured for forms.

Use the narrowest standard method that matches the requirement:

| Requirement | Preferred method |
| --- | --- |
| Rule shared by normal table and form reads | `prepareReadQuery()` |
| Additional full-text behavior | `addFulltextSearchToQuery()` |
| Non-standard column search | `addColumnSearchToQuery()` |
| Non-standard ordering | `addOrderByToQuery()` |
| Form relation selection | Model form-loading methods |
| Table relation selection | Model table-loading methods |

### Extending `prepareReadQuery()`

An override should normally begin by calling
`parent::prepareReadQuery($query, $level, $includeRelations)`. The parent
prepares described columns, lookup values, required joins, and requested
relations. Add only the rule or computed value the model actually requires,
then return the query.

Do not replace `prepareReadQuery()` with a completely independent query unless
you intentionally need to replace standard record loading.

### Search and ordering

Ordinary columns, enums, lookups, numbers, dates, date-times, and booleans
already have standard search behavior.

Override search only for a value that the normal column description cannot
handle, such as a computed SQL alias. The alias must already be part of the
prepared query.

Override ordering only when a displayed value requires special ordering. A
custom ordering branch should:

- recognize a known field,
- allow only `asc` or `desc`,
- return the modified query,
- delegate other fields to the parent.

Do not build column names or SQL expressions directly from unchecked request
values.

## 4. Configuring lookups

Lookup configuration is split between the model and record manager.

Use the model for how a record is displayed:

- `$lookupSqlValue` for the primary label,
- `getLookupDetails()` for optional secondary text,
- `$lookupUrlDetail` for an optional record URL.

Use the record manager for which choices are returned:

- `prepareLookupQuery()` for lookup restrictions,
- `prepareLookupData()` for any additional safe response value.

### Lookup label

`$lookupSqlValue` is an SQL expression. The framework replaces `{{ '{' }}%TABLE%{{ '}' }}`
with the correct table name or alias:

```php
public ?string $lookupSqlValue = "
  concat(
    ifnull({{ '{' }}%TABLE%{{ '}' }}.code, ''),
    ' ',
    ifnull({{ '{' }}%TABLE%{{ '}' }}.name, '')
  )
";
```

Use stable columns and keep the result short. Never insert request data into
this expression.

### Lookup restrictions

When a lookup should show only an established subset, extend the parent query:

```php
public function prepareLookupQuery(string $search): mixed
{
  $query = parent::prepareLookupQuery($search);

  return $query->where(
    $this->table . '.is_active',
    true
  );
}
```

Calling the parent preserves the standard label fields and search behavior.

If a restriction depends on request context, read the value through Hubleto's
router and validate it before using it.

### Lookup response data

Override `prepareLookupData()` only when a specialized input needs an
additional value. Call the parent first and add only explicitly required,
non-sensitive fields.

Lookup responses should not expose complete records, passwords, tokens, or
unrelated personal information.

## 5. Saving related records

The standard record endpoint saves through `recordSave()`. Related records are
processed only when their relation names are included in `saveRelations`.

### Loading is not saving

These settings have different purposes:

| Setting | Purpose |
| --- | --- |
| Model form-loading methods | Determine related data loaded from the backend |
| Form description `includeRelations` | Determine relation data retained in the form payload |
| `saveRelations` | Determine relations processed by `recordSave()` |

The default form description includes all declared relation names. If a custom
form description restricts `includeRelations`, keep every relation the form
edits.

### Enable relation saving from a form

A custom form can add `saveRelations` to its endpoint parameters:

```tsx
getEndpointParams(): any {
  return {
    ...super.getEndpointParams(),
    saveRelations: ['LINES'],
  };
}
```

Calling the parent preserves the standard endpoint parameters.

Use only relation names declared by the model. Loading `LINES` does not
automatically enable saving it.

### Supported relation types

The standard nested-save implementation processes:

- `HAS_MANY`,
- `HAS_ONE`.

A `BELONGS_TO` relation is normally saved by changing its foreign-key lookup
column on the main record.

`HAS_MANY_THROUGH` is not processed by the standard nested-save switch.
Indirect associations should be managed through an appropriate child or
junction model.

### New child records

When a parent and child are created in the same save, the child does not know
the new parent ID. Put the standard placeholder in the child's parent lookup:

```ts
const line = {
  id: -1,
  id_document: {
    _useMasterRecordId_: true,
  },
  description: '',
};
```

The record manager replaces the placeholder with the saved parent ID.

For nested paths, include every level that must be processed. Saving notes
below lines therefore requires both `LINES` and `LINES.NOTES`.

### Deleting child records

Mark a persisted child with `_toBeDeleted_ = true`. The selected child relation
is deleted through `recordDelete()` when the parent is saved. A new unsaved row
can be removed from the frontend collection instead.

The normal delete permission check still applies.

## 6. Validation and normalization

`recordSave()` validates the main record and the selected related records before
writing them.

Standard validation checks:

- required values,
- values rejected by the corresponding column type.

Children marked with `_toBeDeleted_` are not validated as records that will be
saved.

Before an insert or update, normalization:

- removes unknown keys,
- removes virtual columns,
- uses each column's normalization logic,
- applies configured null values where appropriate.

Use `recordSave()` for form payloads and relation-aware saves.

`recordCreate()` is a lower-level insert helper. It is suitable for controlled
internal inserts, but it is not the complete relation-aware validation and save
pipeline.

Model callbacks also participate in these operations. They are covered in
Lesson 11.

## 7. Record permissions

Record permissions are decided by the model's `getPermissions()` method. They
are separate from relation loading and from the application-level permission
manager.

`getPermissions()` returns four Boolean values in a fixed order:

| Position | Meaning |
| --- | --- |
| `0` | Can create |
| `1` | Can read |
| `2` | Can update |
| `3` | Can delete |

A custom model can calculate these values from the current user and the record:

```php
public function getPermissions(array $record): array
{
  return [$canCreate, $canRead, $canUpdate, $canDelete];
}
```

The framework model allows all four operations by default. The ERP model
narrows record access according to the current user's record policies and, when
present, ownership, manager, team, and sharing values. A model can override
`getPermissions()` when it needs a more specific rule.

### Permissions in `recordSave()`

`recordSave()` checks permissions before validation and before the database
write. Standard new form records use a negative ID; `recordSave()` evaluates
their submitted values and checks the create permission. For a positive
existing ID, it loads the original stored record and checks the update
permission against that data.

If the operation is not allowed, `recordSave()` throws a
`NotEnoughPermissionsException` and does not continue with the write.

Selected child records are saved through their own model's `recordSave()` call,
so each child save also uses the permissions of its own model.

Calling `recordCreate()`, `recordUpdate()`, or Eloquent write methods directly
does not run the permission check performed by `recordSave()`. Use the standard
save flow for user-submitted form records unless bypassing it is intentional.

### Permissions in `recordRead()`

`recordRead()` loads one record and calls `getPermissions()` for it. If the read
permission is false, it throws `NotEnoughPermissionsException` instead of
returning the record.

For an allowed record, the returned payload contains `_PERMISSIONS`. Standard
forms use these values to decide whether create, update, and delete actions
should be available.

Table reads use `recordReadMany()`. It checks every loaded row. An unreadable row
is replaced by a payload containing only `_PERMISSIONS`; its record values are
not returned.

### Permissions in `recordDelete()`

`recordDelete()` first reads the stored record through `recordRead()`, so the
read permission is checked. It then checks the delete permission before
executing the delete query. If either check fails, the record is not deleted.

Frontend permission flags improve the interface, but they are not the security
boundary. Save, read, and delete permissions are enforced again by the backend
record methods.

## 8. Defining indexes

Some column types generate their standard indexes. A lookup column, for
example, generates an index for its foreign-key column.

Use the model's `indexes()` method for named indexes and unique constraints:

```php
public function indexes(array $indexes = []): array
{
  return parent::indexes([
    'document_number_unique' => [
      'type' => 'unique',
      'columns' => [
        'document_number' => [
          'order' => 'asc',
        ],
      ],
    ],
  ]);
}
```

The current SQL generator supports `index` and `unique` definitions.

Use:

- a normal index for a real query access pattern,
- a unique constraint when duplicate values would violate the data model,
- a composite index when queries filter or order by the same column sequence.

Column order matters in a composite index.

Changing `indexes()` does not alter an installed database. Add the matching
schema change in a new migration.

## 9. Evolving the schema

Initial migrations use zero-padded names such as `Document_0001.php`. When the
installed schema changes, add the next migration, such as `Document_0002.php`.

The class name must match the filename without `.php`.

### Initial migration vs. follow-up migration

The first migration describes how to create the model's table on a fresh
installation. Its `upgradeSchema()` normally contains `CREATE TABLE` together
with the initial columns and indexes. Its `upgradeForeignKeys()` adds the
initial foreign keys after the required tables exist.

A follow-up migration starts from a database where the earlier migration has
already run. It changes that existing structure with operations such as `ALTER
TABLE ... ADD`, `ALTER TABLE ... MODIFY`, or `ALTER TABLE ... DROP`. It must not
try to create the table again.

Both a fresh installation and an upgraded installation run the migration
sequence in order and should finish with the same schema.

### Never rewrite deployed history

After `_0001` has been released or applied to another installation, treat it as
immutable. Hubleto stores the latest installed schema and foreign-key migration
versions for each model, so an installation that has already applied `_0001`
will not run it again.

Editing it makes fresh installations and upgraded installations produce
different schemas: a fresh installation sees the edited `_0001`, while an
existing installation keeps the result of the old version. Add `_0002` instead
so both installation paths receive the same change.

### Migration methods

Each current migration implements the following four methods:

| Method | Responsibility |
| --- | --- |
| `upgradeSchema()` | Add or change tables, columns, and indexes |
| `upgradeForeignKeys()` | Add foreign-key constraints |
| `downgradeForeignKeys()` | Describe removal of those constraints |
| `downgradeSchema()` | Describe reversal of the schema change |

The standard migration command applies upgrades in two rounds: schema first,
foreign keys second. The current CLI does not expose a rollback command, but
downgrade methods remain required by the migration interface.

### Follow-up migration example

A later migration should alter the existing table rather than recreate it:

```php
class Document_0002 extends Migration
{
  public function upgradeSchema(): void
  {
    $this->db->execute(
      "alter table `documents`
        add `reference` varchar(255) null"
    );
  }

  public function downgradeSchema(): void
  {
    $this->db->execute(
      "alter table `documents`
        drop `reference`"
    );
  }

  public function upgradeForeignKeys(): void
  {
  }

  public function downgradeForeignKeys(): void
  {
  }
}
```

For a new required column, first add it in a state compatible with existing
rows, populate those rows, and then apply the final nullability or default rule.

When adding a lookup column:

1. add the column and index in `upgradeSchema()`,
2. add its constraint in `upgradeForeignKeys()`,
3. reverse those operations in the corresponding downgrade methods.

Lookup columns use `RESTRICT` for `ON DELETE` and `ON UPDATE` unless configured
otherwise.

### CLI commands

```bash
php hubleto create migration "<AppNamespace>" <ModelName>
php hubleto migrate
php hubleto migrate "<AppNamespace>" <ModelName>
php hubleto migrate dry-run
```

The generator creates an initial `_0001` migration from the model. Review the
generated SQL.

For subsequent changes, create the next sequential migration instead of
replacing the initial one.

`dry-run` lists pending schema migrations without applying them. Test the real
migration on a disposable or development database before deployment.

## 10. Recommended implementation workflow

For a parent model with editable child records:

1. Declare the child relation in the parent model.
2. Implement the matching Eloquent relation method.
3. Load the relation for the form when the form uses it.
4. Avoid loading it for tables that do not use it.
5. Keep the relation in the form description.
6. Add it to `saveRelations`.
7. Use `_useMasterRecordId_` for the parent lookup on new children.
8. Create the child lookup column, index, and foreign key in migrations.

Test:

- loading existing children,
- creating the parent and children together,
- updating children,
- deleting persisted children,
- validation errors in child records,
- fresh installation and upgrade to the same final schema.

## 11. Common mistakes

- Loading a larger relation graph than the screen uses.
- Assuming that a loaded relation is automatically saved.
- Listing `LINES.NOTES` without listing `LINES` in `saveRelations`.
- Replacing `prepareLookupQuery()` without calling its parent.
- Exposing complete or sensitive records in lookup responses.
- Treating `recordCreate()` as a complete form-save pipeline.
- Relying only on frontend permission flags instead of backend record checks.
- Changing model or index metadata without a migration.
- Editing an already deployed `_0001` migration.
- Adding foreign-key constraints in `upgradeSchema()` instead of
  `upgradeForeignKeys()`.

## Best practices

- Use the narrowest standard extension point.
- Call parent implementations when extending query or lookup behavior.
- Load fewer relations in tables than in forms.
- Keep relation names consistent across the model, record manager, and form.
- Keep lookup labels and responses compact.
- Use `recordSave()` for relation-aware form saves.
- Save only explicitly selected relations.
- Keep record permission decisions in the model's `getPermissions()` method.
- Use database constraints for real uniqueness rules.
- Treat applied migrations as immutable.
- Test both fresh-installation and upgrade paths.

## Study material

| Resource | Description |
| --- | --- |
| [Lesson 8](lesson-8) | Model, RecordManager, relation, record API, and migration foundations. |
| [Lesson 10](lesson-10) | Description API configuration. |
| Lesson 11 | Model lifecycle callbacks. |
| [Models](../../docs/framework/models) | General model structure. |
| [Record manager](../../docs/framework/models/record-manager) | Eloquent-backed data access. |
| [Records API](../../docs/framework/models/records-api) | Standard record endpoints. |

## Videos

The webinar recording for this lesson will be added after the live session.

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community
portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the
[developer certification course](../../certification).