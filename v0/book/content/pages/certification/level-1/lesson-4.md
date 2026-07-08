# Lesson 4: Custom App Frontend Basics

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

<iframe width="560" height="315" src="https://www.youtube.com/embed/6RQDVTA-yKE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Introduction

In this lesson, we explore how Hubleto handles the frontend layer of custom applications. Following the backend data structures created in Lesson 3, you will learn how to present this data using the **DescriptionAPI**, route requests via **Controllers**, and render fully interactive interfaces using **Twig Views** combined with Hubleto's built-in **React UI components**.

While the video walks you through the practical steps, this guide dives deeper into the architecture of how real ERP apps are built in Hubleto, providing best-practice code snippets and explaining the mechanisms under the hood.

> **What you will learn:**
>
> * How to use the **DescriptionAPI** to configure UI elements (columns, filters, features) directly from your PHP Models.
> * How **Controllers** catch routes and parse URL queries into `$this->viewParams` using `prepareView()`.
> * How to embed generated **React components** into **Twig templates** using typed custom HTML tags (e.g., `<hblreact-car-rental-table-cars>`).
> * How to pass parameters between Twig and React to handle URL-based record routing seamlessly.

## 1. Configuring UI with the DescriptionAPI

The **DescriptionAPI** is Hubleto's elegant solution for keeping your user interface logic tightly coupled with your data models. Instead of writing separate frontend React code for every table and form, you configure how the UI should look directly within your PHP Model classes using the `describeTable()` and `describeForm()` methods.

When the browser requests a page, the React frontend automatically fetches this configuration and dynamically builds the data grids and forms.

### Adding Filters to the Data Grid

If you open `Models/Car.php`, you will find the `describeTable()` method. This method returns a description object that dictates which features of the table are enabled (like search bars, headers, and footers).

While the CLI snippet provides a quick way to add a filter using the `ui['filters']` array, **real ERP apps use the `$description->addFilter()` method** for cleaner, more robust configurations. 

Here is how you add a custom "Archive" filter to the Cars table the proper way:

```php
  /**
   * Returns description of the table showing data from this model.
   */
  public function describeTable(): \Hubleto\Framework\Description\Table
  {
    $description = parent::describeTable();
    $description->ui['addButtonText'] = 'Add Car';
    $description->show(['header', 'fulltextSearch', 'columnSearch', 'moreActionsButton']);
    $description->hide(['footer']);

    // Defining a custom table filter
    $description->addFilter('fArchive', [
      'title' => $this->translate('Archive'),
      'direction' => 'horizontal',
      'options' => [
        0 => $this->translate('Active'),
        1 => $this->translate('Archived')
      ],
      'default' => 0 // Set the default filter state
    ]);

    return $description;
  }
```

Without writing any JavaScript or React code, this instantly creates a fully functional filter dropdown in the table's user interface.

## 2. Controllers: Connecting Routes to Views

In Hubleto, Controllers are lean and act as the bridge between the routing system and your UI. When a user navigates to the `/carrental/cars` URL, the `Controllers/Cars.php` controller is triggered.

Instead of outputting HTML directly, standard Hubleto controllers use the `prepareView()` method. This method calculates variables required by the frontend and specifies which Twig template should be rendered.

```php
namespace Hubleto\App\Custom\CarRental\Controllers;

class Cars extends \Hubleto\Erp\Controller
{
  public function prepareView(): void
  {
    // The parent method handles standard URL parameters
    parent::prepareView();

    // You can pass custom variables to the Twig view here
    $this->viewParams['now'] = date('Y-m-d H:i:s');

    // Specify the Twig template to render
    $this->setView('@Hubleto:App:Custom:CarRental/Cars.twig');
  }
}
```

### The Magic of `parent::prepareView()`

Calling `parent::prepareView()` is critical. It parses the current URL and populates the `$this->viewParams` array with parameters that the React UI expects. For example:
- `viewParams['recordId']`: The ID of the record if specified in the URL (e.g., `/cars/5`).
- `viewParams['q']`: The fulltext search query.
- `viewParams['search']`: Column-specific search rules.
- `viewParams['filters']`: The currently active filters.
- `viewParams['tab']`: The currently active tab in a record's form.

## 3. Views and React Components

Hubleto utilizes **Twig** as its templating engine. Your Twig views form the skeleton of the page, where you mix standard HTML with powerful React components generated by the CLI.

If you open `Views/Cars.twig`, you will see how seamlessly the React data grid is injected into the page:

```html
<hblreact-car-rental-table-cars
  string:tag="table-cars"
  int:record-id="{{ viewParams.recordId }}"
  string:view="{{ viewParams.view }}"
  string:fulltext-search="{{ viewParams.q }}"
  json:column-search="{{ viewParams.search|json_encode }}"
  json:filters="{{ viewParams.filters|json_encode }}"
  string:form-active-tab-uid="{{ viewParams.tab }}"
></hblreact-car-rental-table-cars>
```

### Understanding Custom Elements and Hydration

* **Tag Naming:** In `Loader.tsx`, your React component is registered with a PascalCase name like `CarRentalTableCars`. Hubleto's custom element loader automatically translates this into the kebab-case HTML tag `<hblreact-car-rental-table-cars>`.
* **Property Typing:** Notice prefixes like `int:`, `string:`, and `json:`. HTML attributes are inherently strings, so Hubleto's custom element parser uses these prefixes to cast variables into the correct JavaScript data types before passing them to the React component as props.
* **URL Routing:** By mapping `viewParams` properties (like `int:record-id`) to the React component, deep-linking works instantly. If a user loads `/cars/10`, the controller reads the `10`, passes it to Twig, and Twig passes it to the React table, which automatically opens the modal for record #10.

## Study material

| Resource | Description |
| --- | --- |
| [Description API](../../docs/erp/advanced-development/description-api) | Learn how to describe Tables and Forms using the DescriptionAPI. |
| [Controllers](../../docs/erp/advanced-development/core-classes/controller) | Understand how to handle routing logic and render views. |
| [Views](../../docs/erp/advanced-development/customizing-ui/view) | Learn about the Twig templating engine used in Hubleto. |
| [React UI Components](../../docs/framework/views/react-ui) | How to use Hubleto's built-in React components in your views. |
| [Sample `CarRental` app](https://github.com/mrgopes/hubleto-car-rental) | Source code of the completed CarRental app. |

## Archived livestream

<iframe width="560" height="315" src="https://www.youtube.com/embed/1LeP0r5-JLo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification course](../../certification).