# Lesson 5: Custom App Development - CarRental app

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

## Introduction

In this lesson, we will summarize the custom `CarRental` app created in Lessons 3 and 4. The goal is not to add new architecture concepts yet, but to consolidate what has already been built and explain how the backend and frontend parts fit together in one small Hubleto app.

This lesson is intentionally practical. Open the `CarRental` source code and follow along with the explanations below. By the end, you should be able to clearly explain which files belong to the backend, which files belong to the frontend, and how Hubleto connects them into one working app.

> **What you will learn:**
>
> * How the `CarRental` app is structured after Lessons 3 and 4.
> * Which backend files define the app, models, record managers, and migrations.
> * Which frontend files define routes, controllers, views, and React components.
> * How one request travels from URL to controller, Twig view, React table, form, and model.
> * What mistakes developers most often make at this stage.

## 1. Overview of previous development

So far, we have built a simple custom app called `CarRental`. The app demonstrates the standard Hubleto pattern for creating a custom CRUD application:

1.  Create an app with `php hubleto create app`.
2.  Create models with `php hubleto create model`.
3.  Create UI with `php hubleto create mvc`.
4.  Adjust the generated code to match the real business requirement.

The `CarRental` app contains two custom models:

* `Car`
* `RentalHistory`

These models are managed through standard Hubleto routes, controllers, Twig views, and React table/form components.

At this point, the app already shows the complete basic lifecycle of a Hubleto custom app:

* the app is registered by its `Loader`,
* its models define database structure and UI descriptions,
* migrations create SQL tables,
* controllers prepare view parameters,
* Twig views render `<hblreact-*>` tags,
* React components load model descriptions and data from the backend.

## 2. Recapitulation of backend programming

The backend of the `CarRental` app consists mainly of the following files:

| File | Purpose |
| --- | --- |
| `Loader.php` | Registers routes and installs the app schema. |
| `Models/Car.php` | Defines the `Car` model, columns, relations, and DescriptionAPI settings. |
| `Models/RentalHistory.php` | Defines the `RentalHistory` model. |
| `Models/RecordManagers/Car.php` | Handles Eloquent-level database work for `Car`. |
| `Models/RecordManagers/RentalHistory.php` | Handles Eloquent-level database work for `RentalHistory`. |
| `Models/Migrations/*.php` | Creates and updates SQL tables for the models. |

### `Loader.php`

The app loader is the entry point of the app on the backend side. It:

* registers URLs such as `carrental`, `carrental/cars`, and `carrental/rental-historys`,
* installs model schemas inside `installApp()`,
* optionally integrates with settings, app menu, or calendar services.

This is the place where the app becomes visible to the framework.

### Models

The model files are the core of the backend.

Each model defines:

* the SQL table name,
* the record manager class,
* lookup behavior,
* optional relations,
* column definitions in `describeColumns()`,
* table and form behavior using the DescriptionAPI.

For example, in `Car.php`, the `describeColumns()` method defines which fields exist and how they should be displayed. The `describeTable()` method configures the generated table UI.

### RecordManagers

The record manager is the database-oriented part of the model. It usually:

* extends `\Hubleto\Erp\RecordManager`,
* defines relation methods such as `belongsTo()` or `hasMany()`,
* optionally customizes `prepareReadQuery()` for filtering or joining.

At Level 1, you should understand that:

* **Model** = structure and UI description
* **RecordManager** = actual database query behavior

### Migrations

Each model needs at least one migration to create its table.

The `CarRental` app demonstrates the standard pattern:

* one migration file per model,
* `upgradeSchema()` creates the table,
* `downgradeSchema()` removes it,
* `upgradeForeignKeys()` and `downgradeForeignKeys()` handle constraints when needed.

The migrations are applied when the app is installed or when you run `php hubleto migrate`.

## 3. Recapitulation of frontend programming

The frontend of the `CarRental` app consists mainly of these files:

| File | Purpose |
| --- | --- |
| `Loader.tsx` | Registers React components so Twig can use them. |
| `Controllers/Home.php` | Prepares the welcome screen. |
| `Controllers/Cars.php` | Prepares the cars list screen. |
| `Controllers/RentalHistorys.php` | Prepares the rental history screen. |
| `Views/Home.twig` | Shows links to app sections. |
| `Views/Cars.twig` | Renders the cars table component. |
| `Views/RentalHistorys.twig` | Renders the rental history table component. |
| `Components/TableCars.tsx` | React table component for cars. |
| `Components/FormCar.tsx` | React form component for cars. |
| `Components/TableRentalHistorys.tsx` | React table component for rental history records. |
| `Components/FormRentalHistory.tsx` | React form component for rental history records. |

### Controllers

The controllers are intentionally thin. Their job is to:

* call `parent::prepareView()`,
* read URL parameters into `viewParams`,
* choose the Twig template with `setView()`.

This is the standard Hubleto pattern. Business logic belongs in models or dedicated services, not in controllers.

### Twig views

Twig views are the bridge between PHP and React.

For example, a view such as `Cars.twig` renders a custom element like:

```html
<hblreact-car-rental-table-cars
  int:record-id="{{ viewParams.recordId }}"
></hblreact-car-rental-table-cars>
```

This custom HTML tag does not work by itself. It works only because the matching React component is registered in `Loader.tsx`.

### React components

The `TableCars.tsx` and `FormCar.tsx` files are generated scaffolds that were then adjusted for the app.

The usual pattern is:

* table extends `TableExtended`,
* form extends `FormExtended`,
* table defines the model and returns the form in `renderForm()`,
* form defines tabs and record form URL behavior.

This structure is used across many Hubleto apps and is the standard pattern that new developers should learn first.

## 4. How the whole app works together

To understand Hubleto development, trace one screen from start to finish.

Example: opening `carrental/cars`

1.  `Loader.php` matches the URL to `Controllers\Cars`.
2.  `Controllers\Cars.php` calls `parent::prepareView()` and sets `Cars.twig`.
3.  `Cars.twig` renders `<hblreact-car-rental-table-cars>`.
4.  `Loader.tsx` has already registered `CarRentalTableCars`.
5.  The browser loads `TableCars.tsx`.
6.  `TableCars.tsx` works with model `Hubleto/App/Custom/CarRental/Models/Car`.
7.  The backend model provides DescriptionAPI data and records.
8.  The table displays data and opens `FormCar.tsx` when a record is opened.

This request flow is the core mental model of Level 1 Hubleto development.

## 5. Feedback from developers

At this stage, developers usually make the same few mistakes:

### Mistake 1: Putting too much logic into controllers

Controllers should stay small. If data structure or filtering logic grows, move it into the model or record manager.

### Mistake 2: Editing React first instead of fixing the model

Many table and form changes should be done in the model through the DescriptionAPI, not by rewriting the React UI.

### Mistake 3: Forgetting to register React components

If a Twig `<hblreact-*>` tag does not work, first check `Loader.tsx`.

### Mistake 4: Forgetting to install or migrate the app

If the route exists but the table fails, make sure the app is installed and the schema exists.

### Mistake 5: Keeping unused scaffold code

The CLI generator is useful, but it often creates placeholders that should be cleaned up. Keep only the files and features the app really uses.

## Best practices

* Start with CLI scaffolding, then simplify.
* Keep controllers thin.
* Use models and the DescriptionAPI as the main place for CRUD configuration.
* Use RecordManagers for query-level customization.
* Use Twig only as a view bridge, not as a place for business logic.
* Reuse the same structure that is already used in official Hubleto apps.

## Study material

| Resource | Description |
| --- | --- |
| [Lesson 3](lesson-3) | Backend basics used to build the `CarRental` app. |
| [Lesson 4](lesson-4) | Frontend basics used to build the `CarRental` app. |
| [Models](../../docs/framework/models) | Reference for model structure and DescriptionAPI. |
| [Record manager](../../docs/framework/models/record-manager) | Reference for RecordManagers and Eloquent behavior. |
| [Relations](../../docs/framework/models/relations) | Reference for `belongsTo` and `hasMany`. |
| [Views](../../docs/erp/advanced-development/customizing-ui/view) | Reference for Twig views. |
| [React UI Components](../../docs/framework/views/react-ui) | Reference for Hubleto React UI usage. |
| [Sample `CarRental` app](https://github.com/mrgopes/hubleto-car-rental) | Source code of a sample custom app. |

## Archived livestream

<iframe width="560" height="315" src="https://www.youtube.com/embed/APYWRZ6r3l8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification programme](../../certification).
