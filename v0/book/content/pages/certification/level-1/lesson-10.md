# Lesson 10: Description API

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

## Introduction

Hubleto's **Description API** lets a model describe how its standard tables, forms, and inputs should look and behave.

Instead of repeating common CRUD configuration in PHP views and custom frontend components, you keep reusable field metadata in the model and customize each generated interface through three methods:

* `describeTable()` configures a model's table.
* `describeForm()` configures the create and update form.
* `describeInput()` configures an individual input.

A normal standalone Hubleto installation already contains the framework and UI support needed to use these methods. You do not need separate clones of Hubleto's internal packages. You normally work only in your app's model, record manager, migration, and components.

> **What you will learn:**
>
> * The purpose and basic principles of the Description API.
> * The difference between columns, table columns, and form inputs.
> * Table configuration with `describeTable()`.
> * Form configuration with `describeForm()`.
> * Input configuration with `describeInput()`.
> * How to configure columns, sorting, filters, defaults, enums, and lookups.
> * Which rules belong in the Description API and which must remain in backend logic.

## 1. Basic principles

The Description API is a declarative configuration layer. Your model states what the standard interface should contain, and Hubleto uses that description when displaying the model.

The main objects are:

| Description class | Model method | Responsibility |
| --- | --- | --- |
| `Hubleto\Framework\Description\Table` | `describeTable()` | Table UI options, permissions, columns, sorting, and filters. |
| `Hubleto\Framework\Description\Form` | `describeForm()` | Form UI options, inputs, defaults, permissions, and relation metadata. |
| `Hubleto\Framework\Description\Input` | `describeInput()` | One input's type, label, state, choices, lookup, and component options. |

The Description API should contain presentation metadata and standard interface behavior. It should not contain database-query logic, migrations, or business operations.

Use this separation:

| Concern | Correct location |
| --- | --- |
| Field type and reusable metadata | `describeColumns()` |
| Visible table columns and table controls | `describeTable()` |
| Form controls and defaults | `describeForm()` |
| Input-specific presentation | `describeInput()` |
| Filtering and data loading | Record manager or CRUD controller |
| Record validation and protected values | Model, columns, callbacks, and save logic |
| Database schema | Migration |

### Always start with the parent description

For normal model customization, start with the inherited description, modify it, and return it:

```php
public function describeTable(): \Hubleto\Framework\Description\Table
{
  $description = parent::describeTable();

  // Customize the table here.

  return $description;
}
```

Use the same pattern for forms and inputs:

```php
public function describeForm(): \Hubleto\Framework\Description\Form
{
  $description = parent::describeForm();
  return $description;
}

public function describeInput(
  string $columnName
): \Hubleto\Framework\Description\Input {
  $description = parent::describeInput($columnName);
  return $description;
}
```

The parent methods provide the standard columns, inputs, defaults, relations, permissions, and UI configuration. Returning a new empty description object would discard those inherited settings.

## 2. Columns are the foundation

The Description API builds on the model's `describeColumns()` method.

A column describes more than an SQL field. Depending on its class and configuration, it can provide:

* input type,
* title,
* required and read-only state,
* default value,
* table visibility,
* enum values,
* lookup model,
* numeric precision,
* and optional custom presentation metadata.

Example:

```php
use Hubleto\Framework\Db\Column\Boolean;
use Hubleto\Framework\Db\Column\Currency;
use Hubleto\Framework\Db\Column\Lookup;
use Hubleto\Framework\Db\Column\Text;
use Hubleto\Framework\Db\Column\Varchar;

public function describeColumns(): array
{
  return array_merge(parent::describeColumns(), [
    'name' => (new Varchar($this, $this->translate('Name')))
      ->setRequired()
      ->setDefaultVisible(),

    'code' => (new Varchar($this, $this->translate('Code')))
      ->setRequired()
      ->setDefaultVisible(),

    'id_category' => (new Lookup(
      $this,
      $this->translate('Category'),
      Category::class
    ))->setDefaultVisible(),

    'price' => (new Currency($this, $this->translate('Price')))
      ->setDecimals(2)
      ->setStep(0.01)
      ->setDefaultValue(0)
      ->setDefaultVisible(),

    'status' => (new Varchar($this, $this->translate('Status')))
      ->setEnumValues([
        'draft' => $this->translate('Draft'),
        'active' => $this->translate('Active'),
        'archived' => $this->translate('Archived'),
      ])
      ->setDefaultValue('draft')
      ->setDefaultVisible(),

    'description' => new Text(
      $this,
      $this->translate('Description')
    ),

    'is_featured' => (new Boolean(
      $this,
      $this->translate('Featured')
    ))->setDefaultValue(false),
  ]);
}
```

### Three related collections

It is important to distinguish these collections:

| Collection | Purpose |
| --- | --- |
| Model columns | Complete field definition used by the model and backend record handling. |
| `$tableDescription->columns` | Columns displayed by one table. |
| `$formDescription->inputs` | Inputs available to one form. |

Removing a column from a table description does not remove it from the model or database. This is the correct way to hide a field from one interface without changing the data model.

## 3. Configuring tables with `describeTable()`

`describeTable()` returns `Hubleto\Framework\Description\Table`.

Its most important sections are:

* `$description->ui`
* `$description->permissions`
* `$description->columns`
* `$description->inputs`

### Table UI options

The most commonly used table options are:

| UI key | Purpose |
| --- | --- |
| `title` | Main table title. |
| `addButtonText` | Text displayed in the add-record button. |
| `showHeader` | Shows or hides the complete table header. |
| `showHeaderTitle` | Shows or hides the title inside the header. |
| `showFooter` | Shows or hides the table footer. |
| `showFilter` | Enables the table's filter area. |
| `showSidebarFilter` | Shows or hides the sidebar filter. |
| `showFulltextSearch` | Shows the full-text search control. |
| `showColumnSearch` | Shows search controls for individual columns. |
| `showMoreActionsButton` | Shows the menu containing additional actions. |
| `showAddButton` | Shows or hides the add-record button. |

The `show()` and `hide()` helpers receive names without the `show` prefix:

```php
$description->show(['header', 'fulltextSearch', 'columnSearch']);
$description->hide(['footer', 'addButton']);
```

### Selecting table columns

Use `showOnlyColumns()` when a table should display an explicit set of fields in an explicit order:

```php
$description->showOnlyColumns([
  'name',
  'code',
  'id_category',
  'price',
  'status',
]);
```

Use `hideColumns()` when most inherited columns should remain visible:

```php
$description->hideColumns([
  'description',
  'is_featured',
]);
```

For one column, direct removal is also possible:

```php
unset($description->columns['description']);
```

### Column visibility settings

Reusable table visibility belongs to the column definition:

| Column method | Meaning |
| --- | --- |
| `setAlwaysVisible()` | Keep the column visible. |
| `setAlwaysHidden()` | Keep the column hidden. |
| `setDefaultVisible()` | Show the column by default. |
| `setDefaultHidden()` | Hide it by default while allowing optional display. |
| `setHidden()` | Exclude it from normal table columns. |

Use these methods for reusable defaults. Use `showOnlyColumns()` or `hideColumns()` for a particular table screen.

### Initial sorting

Use `setOrderBy()` to set the table's initial ordering:

```php
$description->setOrderBy('name', 'asc');
```

Common directions are `asc` and `desc`.

The selected field must be available to the table's data query. Sorting by a virtual or calculated field may require matching support in the record manager.

### Table filters

Use `addFilter()` to add a filter control:

```php
$description->addFilter('fStatus', [
  'title' => $this->translate('Status'),
  'direction' => 'horizontal',
  'options' => [
    'draft' => $this->translate('Draft'),
    'active' => $this->translate('Active'),
    'archived' => $this->translate('Archived'),
  ],
  'default' => 'active',
]);
```

Common filter settings are:

| Filter key | Purpose |
| --- | --- |
| `title` | Optional filter heading. |
| `options` | Map of submitted values to displayed labels. |
| `default` | Initial selected value. |
| `direction` | Use `horizontal` for a horizontal button group. |
| `type` | Use `multipleSelectButtons` for multiple selected values. |
| `colors` | Optional map of values to colors. |

The filter description creates the interface control. The record manager or custom CRUD controller must apply the selected value to the data query.
Keep the query implementation in the record manager or CRUD controller, validate accepted filter values, and do not build SQL from unchecked client input.

### Table permissions

Tables use four permission flags:

```php
$description->permissions = [
  'canCreate' => true,
  'canRead' => true,
  'canUpdate' => true,
  'canDelete' => false,
];
```

These flags control the standard interface. For example, `canCreate` affects the add action and `canDelete` affects deletion controls.

They are not a replacement for backend authorization. Record loading, saving, and deletion must still enforce the user's real permissions.

## 4. Configuring forms with `describeForm()`

`describeForm()` returns `Hubleto\Framework\Description\Form`.

Its main sections are:

* `$description->ui`
* `$description->permissions`
* `$description->inputs`
* `$description->defaultValues`
* `$description->includeRelations`

### Basic form configuration

```php
public function describeForm(): \Hubleto\Framework\Description\Form
{
  $description = parent::describeForm();

  $description->ui['title'] = $this->translate('Item');
  $description->ui['subTitle'] = $this->translate('Catalog record');
  $description->ui['saveButtonText'] = $this->translate('Save item');
  $description->ui['addButtonText'] = $this->translate('Create item');
  $description->ui['deleteButtonText'] = $this->translate('Delete item');

  return $description;
}
```

### Form UI options

| UI key | Purpose |
| --- | --- |
| `title` | Main form title. |
| `subTitle` | Secondary form text. |
| `showSaveButton` | Shows or hides the save/create action. |
| `showCopyButton` | Shows or hides the copy action. |
| `showDeleteButton` | Shows or hides the delete action. |
| `saveButtonText` | Text used when saving an existing record. |
| `addButtonText` | Text used when creating a record. |
| `copyButtonText` | Text of the copy action. |
| `deleteButtonText` | Text of the delete action. |
| `headerClassName` | Shared CSS class added to the form header. |

Forms also support the `show()` and `hide()` helpers:

```php
$description->show(['saveButton', 'deleteButton']);
$description->hide(['copyButton']);
```

### Form inputs

The parent form description creates inputs from the model's columns.

Remove an input from one form with `unset()`:

```php
unset($description->inputs['internal_note']);
```

Make one form input read-only:

```php
$description->inputs['code']->setReadonly();
```

The order of `$description->inputs` is the normal display order. You can rebuild the array when a specific order is required:

```php
$inputs = $description->inputs;

$description->inputs = [
  'name' => $inputs['name'],
  'code' => $inputs['code'],
  'id_category' => $inputs['id_category'],
  'price' => $inputs['price'],
  'status' => $inputs['status'],
  'description' => $inputs['description'],
];
```

Reuse inherited input objects instead of creating unrelated replacements. This preserves type, enum, lookup, and validation metadata inherited from the columns.

### Default values

`defaultValues` contains initial values for a new record:

```php
$description->defaultValues['status'] = 'draft';
$description->defaultValues['is_featured'] = false;
```

Use a column default when the value is a reusable field default:

```php
'status' => (new Varchar($this, $this->translate('Status')))
  ->setDefaultValue('draft'),
```

Use `describeForm()` when the default depends on the current form context:

```php
$categoryId = $this->router()->urlParamAsInteger('idCategory');

if ($categoryId > 0) {
  $description->defaultValues['id_category'] = $categoryId;
}
```

Description defaults are convenience values for the form. They are not database constraints and must not be trusted as protected values.

### Relations used by a form

`includeRelations` identifies model relations relevant to the form. Relation names must match relations already declared by the model.

Only include relations the form actually uses. Relation declaration is introduced in [Lesson 8](lesson-8), while relation loading and saving are covered in [Lesson 9](lesson-9).

### Form permissions

Forms use the same permission keys as tables:

```php
$description->permissions['canCreate'] = true;
$description->permissions['canRead'] = true;
$description->permissions['canUpdate'] = true;
$description->permissions['canDelete'] = false;
```

Button visibility and permission are separate settings. Hiding the delete button does not revoke delete permission, and setting `canDelete` to false does not replace backend authorization.

## 5. Configuring inputs with `describeInput()`

`describeInput(string $columnName)` returns the input description for one column.

Use it for input-specific or context-dependent presentation:

```php
public function describeInput(
  string $columnName
): \Hubleto\Framework\Description\Input {
  $description = parent::describeInput($columnName);

  switch ($columnName) {
    case 'name':
      $description->setDescription(
        $this->translate('Use the public item name.')
      );
      break;

    case 'id_category':
      $description->setInputProps([
        'uiStyle' => 'select',
      ]);
      break;
  }

  return $description;
}
```

Always call `parent::describeInput($columnName)` so that the input keeps the metadata defined by its column.

### Common input settings

| Method | Purpose |
| --- | --- |
| `setType($type)` | Selects the input type. Normally inherited from the column class. |
| `setTitle($title)` | Changes the input label. |
| `setReadonly()` | Prevents editing in the standard input. |
| `setRequired()` | Marks the input as required in the interface. |
| `setDescription($text)` | Adds help text below the input. |
| `setIcon($class)` | Adds an icon where supported. |
| `setDecimals($count)` | Sets numeric precision metadata. |
| `setStep($step)` | Sets the numeric input increment. |
| `setDefaultValue($value)` | Sets an initial value. |
| `setEnumValues($values)` | Defines a fixed set of values and labels. |
| `setEnumCssClasses($classes)` | Defines visual classes for enum values. |
| `setPredefinedValues($values)` | Provides suggested text values. |
| `setLookupModel($class)` | Selects the model used by a lookup. |
| `setReactComponent($name)` | Selects a registered custom input component. |
| `setCssClass($class)` | Adds shared CSS classes to a compatible input. |
| `setInputProps($props)` | Passes supported options to the selected input component. |

Prefer settings that are supported by the input component included in your Hubleto installation. Advanced custom properties are useful only when a known component consumes them.

### Common input types

The normal column classes select the appropriate input type automatically.

| Type | Typical input |
| --- | --- |
| `varchar` | Single-line text. |
| `password` | Password input. |
| `text` | Multiline text. |
| `json` | Multiline editor for JSON data. |
| `int` | Integer input. |
| `decimal` | Decimal number input. |
| `currency` | Currency value input. |
| `boolean` | Boolean input. |
| `lookup` | Related-record selector. |
| `color` | Color input. |
| `file` | File input. |
| `image` | Image input. |
| `date` | Date picker. |
| `datetime` | Date and time picker. |
| `time` | Time picker. |

Choose the correct column class in `describeColumns()` instead of changing input types repeatedly in `describeInput()`.

### Enum inputs

Use stable stored keys and translated labels:

```php
$description
  ->setEnumValues([
    'draft' => $this->translate('Draft'),
    'active' => $this->translate('Active'),
    'archived' => $this->translate('Archived'),
  ])
  ->setEnumCssClasses([
    'draft' => 'badge-warning',
    'active' => 'badge-success',
    'archived' => 'badge-secondary',
  ])
;
```

The stored value is the key, such as `active`, not the translated label. This keeps database values independent of the current language.

Use an enum for a closed set of valid choices. Use `setPredefinedValues()` when values are suggestions and arbitrary text remains valid.

### Lookup inputs

A lookup normally gets its related model from the column:

```php
'id_category' => new Lookup(
  $this,
  $this->translate('Category'),
  Category::class
),
```

You can change the standard lookup presentation through supported input props:

```php
$description->setInputProps([
  'uiStyle' => 'select',
]);
```

Common lookup styles include `default`, `select`, `buttons`, and `buttons-vertical`.

The selected related record must still be validated by backend logic. A lookup control does not prove that the user may reference every returned record.

### Input priority

The standard input renderer follows this practical priority:

1.  Enum values select an enum input.
2.  Otherwise, a registered custom component is used when configured.
3.  Otherwise, the input type selects a standard component.

Do not configure both enum values and a custom input component unless the intended priority is clear.

### Required and read-only fields

Description settings control the interface:

```php
$description->setRequired();
$description->setReadonly();
```

For a rule that always applies, define it on the column so every generated interface inherits it:

```php
'code' => (new Varchar($this, $this->translate('Code')))
  ->setRequired()
  ->setReadonly(),
```

Column-level `required` participates in standard backend record validation. `readonly` remains presentation metadata and must not be treated as protection against a manually constructed save request.

Enforce protected values and conditional permissions in backend save logic.

## 6. Common mistakes

### Replacing the inherited description

Returning a new empty description can discard inherited columns, inputs, defaults, permissions, and standard behavior.

Start with the corresponding parent method.

### Removing a model column to hide one interface field

This changes the data model itself. Hide the field in the table or form description instead.

### Defining a filter without applying it

`addFilter()` creates the filter control. The record manager or CRUD controller must apply its value to the query.

### Treating UI permissions as authorization

Hidden buttons and read-only inputs do not secure backend operations. Always enforce authorization during data access and saving.

### Repeating static settings in `describeInput()`

If an enum, default, title, or required rule always applies, define it on the column and inherit it everywhere.

### Loading unnecessary relations

Only include relations used by the form. Large nested records make forms slower and harder to maintain.

### Using unsupported component options

Use `setInputProps()` only for props supported by the selected input component in your Hubleto installation.

## Practical exercise

Create a small model containing:

* required name,
* business code,
* category lookup,
* currency amount,
* status enum,
* description,
* and boolean flag.

Then:

1.  Display five columns in a deliberate order.
2.  Enable full-text and column search.
3.  Sort by name ascending.
4.  Add a status filter and apply it in the record manager.
5.  Customize form titles and button text.
6.  Add a context-dependent category default.
7.  Add input help text.
8.  Verify create and update behavior separately.
9.  Confirm that backend validation and permissions still reject invalid requests.

## Best practices

* Keep reusable field metadata in `describeColumns()`.
* Start normal overrides with the parent description.
* Translate all user-facing labels and choices.
* Keep stored enum keys stable and language-independent.
* Use `showOnlyColumns()` when table order matters.
* Apply filters in the backend query.
* Treat defaults as convenience values, not trusted values.
* Load only relations required by the form.
* Keep backend validation and authorization independent of the interface.
* Use custom frontend components only for behavior the Description API cannot express.

## Study material

| Resource | Description |
| --- | --- |
| [Lesson 7](lesson-7) | Introduction to Hubleto's standard tables, forms, and inputs. |
| [Lesson 8](lesson-8) | Models and record managers used with the Description API. |
| [Models](../../docs/framework/models) | Model structure and description methods. |
| [Columns](../../docs/framework/models/columns) | Column classes and reusable column metadata. |
| [Description API](../../docs/erp/advanced-development/description-api) | Description API reference. |
| [Description API - Tables](../../docs/erp/advanced-development/description-api/table) | Table configuration reference. |
| [Description API - Forms](../../docs/erp/advanced-development/description-api/form) | Form configuration reference. |
| [Records API](../../docs/framework/models/records-api) | Standard record loading and saving. |

## Videos

The webinar recording for this lesson will be added after the live session.

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification course](../../certification).
