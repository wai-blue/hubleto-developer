# Lesson 7: Hubleto ReactUi basics

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

## Introduction

In this lesson, you will learn the basics of the Hubleto ReactUi library and how it is used to build interactive user interfaces inside Hubleto apps. You will connect the backend concepts from previous lessons with the frontend layer: React components, Twig views, table props, form props, and the DescriptionAPI.

ReactUi is the standard UI toolkit used by Hubleto for data-management screens. It provides reusable components such as tables, forms, inputs, modals, and helper components. These components allow developers to build ERP-style apps quickly while keeping a consistent user experience across all Hubleto applications.

> **What you will learn:**
>
> * What Hubleto ReactUi is and why it is used in Hubleto apps.
> * How ReactUi components are registered in `Loader.tsx`.
> * How to render ReactUi components inside Twig views using `<hblreact-*>` tags.
> * How typed HTML attributes are converted to React props.
> * Which properties of `<hblreact-table>` and generated table components are used most often.
> * How the DescriptionAPI controls the table and form look & feel.

## 1. What is Hubleto ReactUi?

Hubleto ReactUi is a React-based component library designed for building data-oriented business applications. In Hubleto, most apps start with a model and then need a table and form for managing records of that model. ReactUi provides this functionality out of the box.

The most commonly used components are:

| Component | Purpose |
| --- | --- |
| `Table` | Generic data grid for browsing, searching, sorting, filtering, and opening records. |
| `TableExtended` | Extended table with additional ERP features such as export, import, column customization, and junction-table helpers. |
| `Form` | Generic form component for editing one record. |
| `FormExtended` | Extended form component with Hubleto ERP-specific behavior and layout helpers. |
| Input components | Components for specific data types, such as varchar, text, boolean, date, lookup, file, image, enum, and others. |

In many simple apps, you do not need to create all React components manually. Hubleto can render a generic table using only a model name. For more advanced apps, you create custom `TableSomething.tsx` and `FormSomething.tsx` components and register them in the app's loader.

## 2. Registering ReactUi components in `Loader.tsx`

Before a React component can be used in a Twig view, it must be registered in the app's TypeScript loader. The loader connects a React component class to a custom HTML tag that Twig can render.

Example registration:

```javascript
import App from '@hubleto/react-ui/core/App';
import TableCars from './Components/TableCars';
import FormCar from './Components/FormCar';

class CarRentalApp extends App {
  init() {
    super.init();
    globalThis.hubleto.registerReactComponent('CarRentalTableCars', TableCars);
    globalThis.hubleto.registerReactComponent('CarRentalFormCar', FormCar);
  }
}
```

The registered component name `CarRentalTableCars` is converted into the HTML tag `<hblreact-car-rental-table-cars>`.

The conversion follows this idea:

| Registered name | Twig tag |
| --- | --- |
| `CarRentalTableCars` | `<hblreact-car-rental-table-cars>` |
| `ContactsTableContacts` | `<hblreact-contacts-table-contacts>` |
| `InvoicesTableInvoices` | `<hblreact-invoices-table-invoices>` |

The `hblreact` prefix tells Hubleto that the HTML element must be replaced by the registered React component.

## 3. Using ReactUi in Twig views

Twig views are the server-rendered skeleton of Hubleto pages. ReactUi components are inserted into these views using custom HTML tags.

### Generic table

For simple model management screens, you can use the globally registered generic table:

```html
<hblreact-table
  string:model="Hubleto/App/Community/Settings/Models/Country"
  int:record-id="{{ viewParams.recordId }}"
></hblreact-table>
```

This is useful when you do not need a custom React table class. Hubleto reads the model description from the backend and builds the table and form automatically.

### Custom app table

For most real apps, you will use a custom table component that was written manually such as:

```html
<hblreact-car-rental-table-cars
  string:tag="table-cars"
  int:record-id="{{ viewParams.recordId }}"
  string:view="{{ viewParams.view }}"
  string:fulltext-search="{{ viewParams.q }}"
  json:column-search='{{ viewParams.search|json_encode }}'
  json:filters='{{ viewParams.filters|json_encode }}'
  string:form-active-tab-uid="{{ viewParams.tab }}"
></hblreact-car-rental-table-cars>
```

This pattern is important because it connects URL parameters prepared by the controller to the React table. It enables deep linking, searching, filters, column search, and opening forms directly by URL.

## 4. Typed attributes and React props

HTML attributes are strings by default. Hubleto uses type prefixes to convert HTML attributes to correctly typed React props.

| Prefix | Converted value | Example |
| --- | --- | --- |
| `string:` | String | `string:tag="table-cars"` |
| `int:` | Integer | `int:record-id="5"` |
| `bool:` | Boolean | `bool:readonly="true"` |
| `json:` | Object or array decoded from JSON | `json:filters='{{ viewParams.filters|json_encode }}'` |

The attribute name is converted to the React prop name. For example, `record-id` becomes `recordId`, and `form-active-tab-uid` becomes `formActiveTabUid`.

This means the following Twig attribute:

```html
int:record-id="{{ viewParams.recordId }}"
```

is passed to the React component as:

```javascript
props.recordId
```

## 5. Most commonly used table properties

The table component accepts many props, but Level 1 developers should be comfortable with the following ones:

| Property | Type | Purpose |
| --- | --- | --- |
| `tag` | String | Unique identifier for the table instance. Useful when multiple tables are rendered on one page. |
| `model` | String | Fully qualified model name used by the generic table to load description and data. |
| `recordId` | Integer | Opens a specific record in the form, usually based on the URL. |
| `view` | String | Passes the current table view or custom view mode. |
| `fulltextSearch` | String | Initial fulltext search query. Usually loaded from `viewParams.q`. |
| `columnSearch` | JSON object | Initial column-specific search values. |
| `filters` | JSON object | Initial custom filters. |
| `formActiveTabUid` | String | Opens a specific form tab. |
| `readonly` | Boolean | Disables table-level modifications such as creating and deleting records. Form-level read-only behavior may need to be handled separately. |
| `itemsPerPage` | Integer | Sets the number of records shown on one page. |

Typical usage in Twig:

```html
<hblreact-table
  string:tag="settings-countries"
  string:model="Hubleto/App/Community/Settings/Models/Country"
  int:record-id="{{ viewParams.recordId }}"
  string:fulltext-search="{{ viewParams.q }}"
  json:column-search='{{ viewParams.search|json_encode }}'
  json:filters='{{ viewParams.filters|json_encode }}'
  string:form-active-tab-uid="{{ viewParams.tab }}"
  int:items-per-page="25"
></hblreact-table>
```

## 6. Custom table components

A custom table component usually extends `TableExtended`. It defines default properties and connects the table to a form component.

Example:

```javascript
import React from 'react';
import TableExtended, { TableExtendedProps, TableExtendedState } from '@hubleto/react-ui/ext/TableExtended';
import FormCar from './FormCar';

interface TableCarsProps extends TableExtendedProps {}
interface TableCarsState extends TableExtendedState {}

export default class TableCars extends TableExtended<TableCarsProps, TableCarsState> {
  static defaultProps = {
    ...TableExtended.defaultProps,
    formUseModalSimple: true,
    model: 'Hubleto/App/Custom/CarRental/Models/Car',
  };

  renderForm(): JSX.Element {
    let formProps = this.getFormProps();
    return <FormCar {...formProps}/>;
  }
}
```

When you extend `TableExtended`, the usual way to connect a custom form is to override `renderForm()` and return your form component with `this.getFormProps()`.

This approach is used when you want to:

* Add custom React behavior.
* Use a custom form component.
* Pass custom props from Twig.
* Add custom actions, custom row rendering, or custom callbacks.
* Reuse the same model in different UI contexts.

## 7. Customizing table look & feel with DescriptionAPI

The DescriptionAPI is the preferred way to configure how tables and forms look and behave. Instead of hardcoding UI settings directly in React, you describe the UI in the PHP model.

Example `describeTable()` method:

```php
public function describeTable(): \Hubleto\Framework\Description\Table
{
  $description = parent::describeTable();

  $description->ui['title'] = $this->translate('Cars');
  $description->ui['addButtonText'] = $this->translate('Add Car');
  $description->show(['header', 'fulltextSearch', 'columnSearch', 'moreActionsButton']);
  $description->hide(['footer']);

  return $description;
}
```

These settings are loaded by the React table from the backend. The table then renders itself according to the description.

Common table UI options:

| DescriptionAPI option | Purpose |
| --- | --- |
| `title` | Table title shown in the header. |
| `subTitle` | Smaller text below or near the title. |
| `addButtonText` | Text of the button used for creating new records. |
| `showHeader` | Shows or hides the table header. |
| `showHeaderTitle` | Shows or hides the title in the header. |
| `showFooter` | Shows or hides the footer. |
| `showFilter` | Shows or hides the filter area. |
| `showSidebarFilter` | Shows or hides the sidebar filter area. |
| `showFulltextSearch` | Enables fulltext search. |
| `showColumnSearch` | Enables column-level search inputs. |
| `showMoreActionsButton` | Shows additional table actions. |
| `showAddButton` | Shows or hides the add button. |

## 8. Adding filters with DescriptionAPI

Filters are also configured in `describeTable()`. They are passed to the React table and displayed in the filter UI or sidebar filter area.

Example:

```php
$description->addFilter('fArchive', [
  'title' => $this->translate('Archive'),
  'direction' => 'horizontal',
  'options' => [
    0 => $this->translate('Active'),
    1 => $this->translate('Archived'),
  ],
  'default' => 0,
]);
```

The React table receives the filter values through props and sends them to the backend when data is loaded. This allows the backend model or record manager to modify the query according to the selected filter.

## 9. Customizing forms with DescriptionAPI

Forms are also controlled by model descriptions. You can define basic form UI text such as title and subtitle, and the form also receives the model inputs, permissions, default values, and relations prepared by `parent::describeForm()`.

Example:

```php
public function describeForm(): \Hubleto\Framework\Description\Form
{
  $description = parent::describeForm();

  $description->ui['title'] = $this->translate('Record');
  $description->ui['subTitle'] = $this->translate('Detail');

  return $description;
}
```

The form description is loaded by the React form from the backend. The React form then renders itself according to the description and the model inputs returned by `parent::describeForm()`.

When possible, keep general UI behavior in the DescriptionAPI and use custom React code only for behavior that cannot be described declaratively.

## Best practices

* Use the generic `<hblreact-table>` for simple admin or settings screens.
* Use custom table and form components for real business apps.
* For pages that should support URL-based state, pass `recordId`, `view`, `fulltextSearch`, `columnSearch`, `filters`, and `formActiveTabUid` from `viewParams`.
* Configure table and form look & feel in the PHP model using the DescriptionAPI.
* Keep custom React code focused on special behavior, not standard CRUD features.
* Use typed attributes (`int:`, `string:`, `bool:`, `json:`) to avoid type-related bugs.

## Study material

| Resource | Description |
| --- | --- |
| [Hubleto React UI](../../docs/framework/views/react-ui) | Introduction to Hubleto ReactUi and how to use React components in Twig views. |
| [Description API](../../docs/erp/advanced-development/description-api) | Learn how backend descriptions control UI behavior. |
| [Description API - Tables](../../docs/erp/advanced-development/description-api/table) | Table-specific DescriptionAPI options. |
| [Views](../../docs/erp/advanced-development/customizing-ui/view) | How Twig views are used in Hubleto apps. |
| [Components](../../docs/erp/advanced-development/customizing-ui/component) | How custom React components are structured and used. |
| [Sample `CarRental` app](https://github.com/mrgopes/hubleto-car-rental) | Source code of a sample custom app used in previous lessons. |
| [Hubleto React UI source code](https://github.com/hubleto/react-ui) | Source code of the ReactUi component library. |

## Archived livestream

<iframe width="560" height="315" src="https://www.youtube.com/embed/UQclL0OKL3A" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification course](../../certification).
