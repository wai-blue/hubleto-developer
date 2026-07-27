# Lesson 6: Custom App Development - IpInfoTest app

<i class="fas fa-medal mr-2"></i> Developer Certification Level 1

<iframe width="560" height="315" src="https://www.youtube.com/embed/2CVe1o2NKik" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Introduction

In this lesson, you will build a small custom Hubleto app called `IpInfoTest`. The goal is to combine everything learned in Lessons 3, 4, and 5 into one simple, realistic exercise:

* create an app,
* create a model,
* generate MVC scaffolding,
* edit the generated files,
* connect a public API,
* save data into a custom model,
* display the saved records with the standard Hubleto table and form pattern.

We strongly encourage you to follow along in your own local project and build the app step by step while reading this guide.

> **What you will learn:**
>
> * How to create a second custom app after `CarRental`.
> * How to start from Hubleto CLI scaffolding and then simplify it.
> * How to create a small custom model used only by one app.
> * How to implement a simple controller that talks to an external API.
> * How to reuse generated React table and form components for a favorites list.
> * How to install the app, run migrations, and compile assets.

## Certification task

Finish this task to get the Level-1 certification.

### Task

Create a simple Hubleto app of the `Custom` type named `IpInfoTest` (i.e., namespace `Hubleto\App\Custom\IpInfoTest`). Publish the source code of the app as a public GitHub repository named `hubleto-ip-info-test`.

App functions:

1.  Obtain an IP address from the user for which they want to retrieve detailed information.
2.  Retrieve information about the IP address from any public external API.
3.  Display the retrieved information to the user in a clear format.
4.  Allow the user to save the retrieved information to a list of favorite IP addresses (a custom model must be created).
5.  Allow the user to browse the list of favorite IP addresses, including the saved information.

*Bonus #1: For the last point, use the React components Table and Form. (examples of use are in the Hubleto ERP source code)*

*Bonus #2: Display statistics of the favorite IP addresses, e.g., the count of IP addresses based on their timezone.*

## 1. Preparation

Before starting, make sure you understand the previous lessons:

* [Lesson 3](lesson-3) explains models, record managers, and migrations.
* [Lesson 4](lesson-4) explains controllers, views, and React table/form components.
* [Lesson 5](lesson-5) summarizes the structure of the `CarRental` app.

The `IpInfoTest` app is intentionally much smaller than `CarRental`. This keeps the focus on the standard workflow.

## 2. Step 1: Create the app

Start in your Hubleto project root and create the app scaffold:

```bash
php hubleto create app IpInfoTest
```

This command creates the basic app shell, including:

* `Loader.php`
* `Loader.tsx`
* `manifest.yaml`
* `Controllers/Home.php`
* `Views/Home.twig`

## 3. Step 2: Create the model

The app needs one custom model for storing favorite IP addresses. Generate it with:

```bash
php hubleto create model IpInfoTest FavoriteIp
```

This command creates:

* `Models/FavoriteIp.php`
* `Models/RecordManagers/FavoriteIp.php`
* `Models/Migrations/FavoriteIp_0001.php`

## 4. Step 3: Create the MVC files for the model

Generate the table, form, controller, and view for the favorites list:

```bash
php hubleto create mvc IpInfoTest FavoriteIp
```

This command creates:

* `Components/TableFavoriteIps.tsx`
* `Components/FormFavoriteIp.tsx`
* `Controllers/FavoriteIps.php`
* `Views/FavoriteIps.twig`

It also updates:

* `Loader.tsx`
* `Loader.php`
* `Views/Home.twig`

## 5. Step 4: Keep the app minimal

After scaffolding, remove the generated files that are not needed for this lesson:

* `Controllers/Settings.php`
* `Views/Settings.twig`
* `Calendar.php`
* `Extendibles/AppMenu.php`

The lesson app does not use a custom settings screen, app menu integration, or custom calendar.

## 6. Step 5: Adjust the app manifest and routes

In `manifest.yaml`, change only the app metadata that matters:

```yaml
rootUrlSlug: ip-info-test
name: IpInfoTest
icon: fas fa-network-wired
highlight: Simple custom app for IP lookup and favorite IP addresses.
```

In `Loader.php`, the important part is the route definition and schema install:

```php
$this->router()->get([
  '/^ip-info-test\/?$/' => Controllers\Home::class,
  '/^ip-info-test\/favorite-ips\/add\/?$/' => ['controller' => Controllers\FavoriteIps::class, 'vars' => ['recordId' => -1]],
]);

$this->router()->get([
  '/^ip-info-test\/favorite-ips(\/(?<recordId>\d+))?\/?$/' => Controllers\FavoriteIps::class
]);
```

And in `installApp()`:

```php
$this->getModel(Models\FavoriteIp::class)->upgradeSchema();
```

This is enough to register the main screen, the favorites list, and the favorites form route.

## 7. Step 6: Implement the `Home` controller

The main app screen is not a table. It is a small lookup form, so the main custom logic lives in `Controllers/Home.php`.

The controller does four things:

1.  reads the IP from the request,
2.  validates it,
3.  loads data from a public API,
4.  saves the result into the `FavoriteIp` model.

The important part of `prepareView()` is:

```php
$this->viewParams = [
  'ip' => trim((string) ($_GET['ip'] ?? '')),
  'result' => null,
  'success' => '',
  'error' => '',
];

if ($_SERVER['REQUEST_METHOD'] === 'POST' && (string) ($_POST['action'] ?? '') === 'save') {
  $this->viewParams['ip'] = trim((string) ($_POST['ip'] ?? ''));
  $lookup = $this->lookupIp($this->viewParams['ip']);

  if (!$lookup['ok']) {
    $this->viewParams['error'] = $lookup['error'];
  } else {
    $this->saveFavoriteIp($lookup['data']);
    $this->viewParams['result'] = $lookup['data'];
    $this->viewParams['success'] = $this->translate('IP information saved to favorites.');
  }
} elseif ($this->viewParams['ip'] !== '') {
  $lookup = $this->lookupIp($this->viewParams['ip']);

  if (!$lookup['ok']) {
    $this->viewParams['error'] = $lookup['error'];
  } else {
    $this->viewParams['result'] = $lookup['data'];
  }
}
```

The validation helper stays simple:

```php
private function lookupIp(string $ip): array
{
  if (!$this->isValidIp($ip)) {
    return [
      'ok' => false,
      'error' => $this->translate('Please enter a valid IP address.'),
    ];
  }

  return $this->fetchIpInfo($ip);
}
```

The API request itself is also small:

```php
$ch = curl_init('https://ipwho.is/' . rawurlencode($ip));
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 5);
curl_setopt($ch, CURLOPT_TIMEOUT, 10);

$response = curl_exec($ch);
$curlError = curl_error($ch);
$httpCode = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);
```

And saving to favorites is just one model create call:

```php
$mFavoriteIp->record->create([
  'ip' => (string) ($data['ip'] ?? ''),
  'country' => (string) ($data['country'] ?? ''),
  'region' => (string) ($data['region'] ?? ''),
  'city' => (string) ($data['city'] ?? ''),
  'timezone' => (string) ($data['timezone'] ?? ''),
  'org' => (string) ($data['org'] ?? ''),
  'raw_data' => (string) ($data['raw_data'] ?? ''),
  'queried_at' => date('Y-m-d H:i:s'),
]);
```

For the favorites screen, `Controllers/FavoriteIps.php` stays intentionally minimal:

```php
public function prepareView(): void
{
  parent::prepareView();
  $this->setView('@Hubleto:App:Custom:IpInfoTest/FavoriteIps.twig');
}
```

## 8. Step 7: Define the `FavoriteIp` model

The `FavoriteIp` model is the app's custom database structure.

In `Models/FavoriteIp.php`, keep only the fields you really need:

```php
public string $table = 'ip_info_test_favorite_ips';
public string $recordManagerClass = RecordManagers\FavoriteIp::class;

public ?string $lookupSqlValue = '{{ "{%" }}TABLE{{ "%}" }}.ip';
public ?string $lookupUrlAdd = 'ip-info-test/favorite-ips/add';
public ?string $lookupUrlDetail = 'ip-info-test/favorite-ips/{{ "{%" }}ID{{ "%}" }}';
```

The columns are straightforward:

```php
'ip' => (new Varchar($this, $this->translate('IP address')))->setRequired()->setDefaultVisible(),
'country' => (new Varchar($this, $this->translate('Country')))->setDefaultVisible(),
'region' => (new Varchar($this, $this->translate('Region')))->setDefaultVisible(),
'city' => (new Varchar($this, $this->translate('City')))->setDefaultVisible(),
'timezone' => (new Varchar($this, $this->translate('Timezone')))->setDefaultVisible(),
'org' => (new Varchar($this, $this->translate('Organization')))->setDefaultVisible(),
'raw_data' => (new Text($this, $this->translate('Raw data'))),
'queried_at' => (new DateTime($this, $this->translate('Queried at')))->setRequired()->setDefaultVisible(),
```

And the favorites table should not allow manual record creation:

```php
$description->permissions['canCreate'] = false;
$description->show(['header', 'fulltextSearch', 'columnSearch', 'moreActionsButton']);
$description->hide(['footer']);
```

## 9. Step 8: Define the record manager and migration

In `Models/RecordManagers/FavoriteIp.php`, the record manager is intentionally minimal:

```php
class FavoriteIp extends \Hubleto\Erp\RecordManager
{
  public $table = 'ip_info_test_favorite_ips';
}
```

In `Models/Migrations/FavoriteIp_0001.php`, the important part is the `upgradeSchema()` SQL:

```php
$this->db->execute("set foreign_key_checks = 0;
drop table if exists `ip_info_test_favorite_ips`;
set foreign_key_checks = 1;");
$this->db->execute("SET foreign_key_checks = 0;
create table `ip_info_test_favorite_ips` (
 `id` int(8) primary key auto_increment,
 `ip` varchar(255) ,
 `country` varchar(255) ,
 `region` varchar(255) ,
 `city` varchar(255) ,
 `timezone` varchar(255) ,
 `org` varchar(255) ,
 `raw_data` text ,
 `queried_at` datetime ,
 index `id` (`id`),
 index `ip` (`ip`),
 index `timezone` (`timezone`),
 index `queried_at` (`queried_at`)) ENGINE = InnoDB;
SET foreign_key_checks = 1;");
```

That is enough for this lesson.

## 10. Step 9: Create the main Twig view

The app homepage is implemented in `Views/Home.twig`.

The lookup form itself is plain Twig:

```html
<form method="get">
  <div class="mb-2">
    <label class="form-label" for="ip">{{ "{{" }} translate('IP address') {{ "}}" }}</label>
    <input class="form-control" id="ip" type="text" name="ip" value="{{ "{{" }} viewParams.ip|default('') {{ "}}" }}" placeholder="8.8.8.8" required />
  </div>

  <button class="btn btn-primary" type="submit">{{ "{{" }} translate('Lookup') {{ "}}" }}</button>
</form>
```

The results block is also plain HTML:

```html
{{ "{%" }} if viewParams.result is not empty {{ "%}" }}
  <div class="card p-3 mb-3">
    <table class="table table-sm mb-3">
      <tbody>
        <tr><th>{{ "{{" }} translate('IP address') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.ip {{ "}}" }}</td></tr>
        <tr><th>{{ "{{" }} translate('Country') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.country {{ "}}" }}</td></tr>
        <tr><th>{{ "{{" }} translate('Region') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.region {{ "}}" }}</td></tr>
        <tr><th>{{ "{{" }} translate('City') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.city {{ "}}" }}</td></tr>
        <tr><th>{{ "{{" }} translate('Timezone') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.timezone {{ "}}" }}</td></tr>
        <tr><th>{{ "{{" }} translate('Organization') {{ "}}" }}</th><td>{{ "{{" }} viewParams.result.org {{ "}}" }}</td></tr>
      </tbody>
    </table>
```

And saving the result uses a small POST form:

```html
<form method="post">
  <input type="hidden" name="action" value="save" />
  <input type="hidden" name="ip" value="{{ "{{" }} viewParams.result.ip {{ "}}" }}" />
  <button class="btn btn-success" type="submit">{{ "{{" }} translate('Save to favorites') {{ "}}" }}</button>
</form>
```

This page is a good example of not forcing React where plain Twig is enough.

## 11. Step 10: Create the favorites list view

The saved records are browsed through standard Hubleto CRUD UI.

In `Views/FavoriteIps.twig`, the essential code is:

```html
<hblreact-ip-info-test-table-favorite-ips
  string:tag="table-favorite-ips"
  int:record-id="{{ "{{" }} viewParams.recordId {{ "}}" }}"
  string:view="{{ "{{" }} viewParams.view {{ "}}" }}"
  string:fulltext-search='{{ "{{" }} viewParams.q {{ "}}" }}'
  json:column-search='{{ "{{" }} viewParams.search|json_encode {{ "}}" }}'
  json:filters='{{ "{{" }} viewParams.filters|json_encode {{ "}}" }}'
  string:form-active-tab-uid='{{ "{{" }} viewParams.tab {{ "}}" }}'
></hblreact-ip-info-test-table-favorite-ips>
```

This is the standard pattern:

* controller prepares `viewParams`
* Twig passes them to the React table
* the React table loads data from the model

## 12. Step 11: Adjust the generated React components

The generated React files are a good starting point, but they need a few small edits.

In `Loader.tsx`, register the generated table component:

```typescript
import TableFavoriteIps from './Components/TableFavoriteIps';

globalThis.hubleto.registerReactComponent('IpInfoTestTableFavoriteIps', TableFavoriteIps);
```

In `Components/TableFavoriteIps.tsx`, keep the important defaults:

```typescript
static defaultProps = {
  ...TableExtended.defaultProps,
  formUseModalSimple: true,
  model: 'Hubleto/App/Custom/IpInfoTest/Models/FavoriteIp',
}
```

Fix the URL used when a record form opens:

```typescript
setRecordFormUrl(id: number) {
  window.history.pushState(
    {},
    "",
    globalThis.hubleto.config.projectUrl + '/ip-info-test/favorite-ips/' + (id > 0 ? id : 'add')
  );
}
```

And return the custom form:

```typescript
renderForm(): JSX.Element {
  return <FormFavoriteIp {...this.getFormProps()}/>;
}
```

In `Components/FormFavoriteIp.tsx`, keep only one basic tab:

```typescript
tabs: [
  { uid: 'default', title: <b>{this.translate('Favorite IP')}</b> },
]
```

Set the correct record URL:

```typescript
getRecordFormUrl(): string {
  return 'ip-info-test/favorite-ips/' + (this.state.record.id > 0 ? this.state.record.id : 'add');
}
```

And show the IP address in the title:

```typescript
renderTitle(): JSX.Element {
  const R = this.state.record;
  return <>
    <small>{this.translate('Favorite IP')}</small>
    <h2>{R.ip ?? this.translate('New')}</h2>
  </>;
}
```

## 13. Step 12: Install the app and run migrations

When the files are ready, install the app:

```bash
php hubleto app install "\Hubleto\App\Custom\IpInfoTest" force
```

Then run the model migration:

```bash
php hubleto migrate "\Hubleto\App\Custom\IpInfoTest" FavoriteIp
```

## 14. Step 13: Compile frontend assets

After editing `Loader.tsx`, `TableFavoriteIps.tsx`, and `FormFavoriteIp.tsx`, the browser needs an updated JavaScript bundle.

```bash
npm run build
```

Run this command in the frontend asset project used by your Hubleto installation.

## 15. Final file overview

After all edits, the important files of the final app are:

| File | Purpose |
| --- | --- |
| `manifest.yaml` | App metadata and URL slug |
| `Loader.php` | Backend routes and schema installation |
| `Loader.tsx` | React component registration |
| `Controllers/Home.php` | IP lookup flow and save action |
| `Controllers/FavoriteIps.php` | Favorites list screen |
| `Models/FavoriteIp.php` | Favorite IP model |
| `Models/RecordManagers/FavoriteIp.php` | Record manager |
| `Models/Migrations/FavoriteIp_0001.php` | SQL schema |
| `Views/Home.twig` | Lookup form and result display |
| `Views/FavoriteIps.twig` | React favorites table view |
| `Components/TableFavoriteIps.tsx` | Favorites data grid |
| `Components/FormFavoriteIp.tsx` | Favorites record form |

## Best practices

* Start from Hubleto CLI scaffolding, then simplify.
* Keep the app scope small.
* Remove generated files that the app does not actually use.
* Use a plain Twig form when React is not necessary.
* Use generated table and form components for standard CRUD screens.
* Keep custom controller logic compact and easy to read.
* Reuse standard Hubleto routes, model patterns, and UI structure.

## Study material

| Resource | Description |
| --- | --- |
| [Lesson 3](lesson-3) | Backend basics used in this app. |
| [Lesson 4](lesson-4) | Frontend basics used in this app. |
| [Lesson 5](lesson-5) | Recap of the `CarRental` app structure. |
| [`php hubleto` CLI agent](../../cli-agent) | Commands for creating apps, models, migrations, and MVC files. |
| [Models](../../docs/framework/models) | Model structure and DescriptionAPI basics. |
| [Record manager](../../docs/framework/models/record-manager) | RecordManagers and query customization. |
| [Migrations](../../docs/framework/models/migrations) | How Hubleto migrations work. |
| [Controllers](../../docs/erp/advanced-development/core-classes/controller) | Controller basics. |
| [Views](../../docs/erp/advanced-development/customizing-ui/view) | Twig views in Hubleto. |
| [React UI Components](../../docs/framework/views/react-ui) | Hubleto React UI library usage. |

## Archived livestream

<iframe width="560" height="315" src="https://www.youtube.com/embed/vmbks-cwukQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Do you have any questions?

Do you have any questions or comments? Leave us a message in the community portal.

<a class="btn" href="https://community.hubleto.eu/d/35-qa-developer-certification-level-1"><span class="text">Go to community.hubleto.eu</span></a>

If you are new here, learn more about the [developer certification programme](../../certification).
