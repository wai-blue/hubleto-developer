# Core architecture

Diagram below illustrates the overal architecture of the Hubleto platform.

```html
╔════════════════════════════════════════════════════════════════════════════════════════╗
║ Hubleto platform                                                                       ║
║                                                                                        ║
║ ┌────────────────────────────────────────────────────────────────────────────────────┐ ║
║ │ Hubleto Core                                                                       │ ║
║ │────────────────────────────────────────────────────────────────────────────────────│ ║
║ │ - User management, App management, Settings management                             │ ║
║ │ - MVC, Routing, Translations, Authentication, Permissions                          │ ║
║ │ - Automation tools                                                                 │ ║
║ │ - Community apps                                                                   │ ║
║ └────────────────────────────────────────────────────────────────────────────────────┘ ║
║    ↓                                                                                   ║
║ ┌────────────────────────────────────────────────────────────────────────────────────┐ ║
║ │ Hubleto App (your development world)                                               │ ║
║ │────────────────────────────────────────────────────────────────────────────────────│ ║
║ │ Loader, Controllers, Models, Views, Translations                                   │ ║
║ └────────────────────────────────────────────────────────────────────────────────────┘ ║
║    ↓                                                                                   ║
║ ┌────────────────────────────────────────────────────────────────────────────────────┐ ║
║ │ Hubleto Framework                                                                  │ ║
║ │────────────────────────────────────────────────────────────────────────────────────│ ║
║ │ Application front-end                    │ Application back-end                    │ ║
║ │ (Components, Views)                      │ (Controllers, Models, Database)         │ ║
║ │ with React, TailwindCSS, Primereact      │ with Symfony, Twig, Eloquent            │ ║
║ └────────────────────────────────────────────────────────────────────────────────────┘ ║
╚════════════════════════════════════════════════════════════════════════════════════════╝
```

## Hubleto platform

Hubleto platform provides a set of features to **manage a complex project** built from various apps and to simplify application development.

These features include:

  * User management, App management, Settings manager
  * Foundation for MVC, routing, language translations, user authentication and permissions management
  * Automation tools
  * Repository of community apps

All these features are made available to you right after you run `php hubleto init` in your console (see more about [CLI agent](../cli-agent)) in your project's root folder.

```
php hubleto init # command to initialize Hubleto project
```

## Customizing Hubleto Core

When you initialize your Hubleto project, all Hubleto Core features will be available to you. However, there can be numerous situations in which you might need different implementation of one these features and so **all Hubleto Core features are fully customizable**.

In most cases, customizations are implemented in two steps:

  * implement your custom, fully separated class
  * modify `\Hubleto\Erp` class in `src/Main.php` to use your new class.

## Default features

In following chapters, we will learn how default features of Hubleto are implemented and we will provide examples how you may customize them for your project.

<div class="mt-8 grid gap-8 md:grid-cols-2">
  <div class="card border-yellow-300">
    <div class="card-header bg-yellow-800 text-yellow-200">Hubleto Core</div>
    <div class="card-body flex flex-col gap-2">
      <a href="#" class="btn btn-white block"><span class="text">User interface</span></a>
      <a href="core-architecture/user-management-and-authentication" class="btn btn-white block"><span class="text">User management and authentication</span></a>
      <a href="#" class="btn btn-white block"><span class="text">App management</span></a>
      <a href="#" class="btn btn-white block"><span class="text">Settings management</span></a>
      <a href="#" class="btn btn-white block"><span class="text">Automation tools</span></a>
    </div>
  </div>
  <div class="card border-green-300">
    <div class="card-header bg-green-900 text-green-200">Hubleto App</div>
    <div class="card-body flex flex-col gap-2">
      <a href="#" class="btn btn-white block"><span class="text">Loader</span></a>
      <a href="#" class="btn btn-white block"><span class="text">Models, views & controllers</span></a>
      <a href="#" class="btn btn-white block"><span class="text">Routing</span></a>
      <a href="#" class="btn btn-white block"><span class="text">Translations</span></a>
      <a href="#" class="btn btn-white block"><span class="text">User permissions</span></a>
    </div>
  </div>
</div>