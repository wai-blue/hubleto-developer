# Level 1

Become a certified Hubleto developer.

The first level of certification ensures that the certified developer knows the basics of Hubleto
development process, its architecture, built-in functionality and most commonly used software
design patterns like, e.g. routing, controllers or views.

<a class="btn btn-primary btn-large" href="level-1/webinars-dates-topics-and-links">
  <span class="icon"><i class="fas fa-calendar"></i></span>
  <span class="text">Full list of all webinars with dates and topics</span>
</a>

> You will learn how to:
> 
>   * prepare the development environment
>   * deploy Hubleto to customer's server
>   * create custom apps
>   * define models, views and controllers
>   * install custom app with default data

## Introduction

This document contains training material for Hubleto Certified Developer training. It is a source of questions and answers for the certification course.

To get a Level 1 certification, you shall pass the certification course. Here you will find training materials and resources to pass this course.

## Prerequisites

You shall have the following knowledge:

  * At least junior level of PHP, Javascript, SQL, HTML and CSS.
  * Optionally, junior level of React / Typescript \- this is required only if a developer is going to implement react-based functionality.
  * Experience with `composer` \- PHP’s package manager and `npm` \- Javascript’s package manager.
  * Experience with setting up a local development environment \- a webserver (Apache or Nginx) with PHP and SQL database (MariaDB or MySQL).
  * Good understanding of the MVC software design pattern.

Having this knowledge, a developer shall be able to configure his local development environment as described in [this guide](../getting-started).

## Resources

Before starting, check out these additional resources:

* [https://www.hubleto.eu](https://www.hubleto.eu)  
* [https://github.com/hubleto](https://github.com/hubleto)  
* [https://developer.hubleto.eu](https://developer.hubleto.eu)  
* [https://help.hubleto.eu](https://help.hubleto.eu)  
* [https://community.hubleto.eu](https://community.hubleto.eu)  
* [https://www.linkedin.com/company/hubleto](https://www.linkedin.com/company/hubleto)  
* [https://www.reddit.com/r/hubleto/](https://www.reddit.com/r/hubleto/)
* [https://github.com/mrgopes/hubleto-car-rental](https://github.com/mrgopes/hubleto-car-rental)

## Lessons

The certification course is composed of several lessons. Each lesson covers a specific topic like, e.g. basic introduction to Hubleto or best practices in development. 

## Lesson #1: Introduction to Hubleto
  - About Hubleto.
  - Install production ready Hubleto using Composer.
  - Overview of Hubleto apps and basic development tools.

<a class="btn" href="level-1/lesson-1"><span class="text">Open lesson #1</span></a> <span class="inline-flex items-center gap-2 align-middle ml-2 rounded-full border border-blue-200/25 bg-blue-400/5 px-2 py-1 text-xs font-medium leading-none text-blue-200/90"><i class="fas fa-video text-[0.7rem]"></i><span>Video available</span></span>

## Lesson #2: Overview of Hubleto features
  - CRM, Marketing, Sales, Productivity, Finance, Maintenance, Help
  - Users, settings, platform config and themes
  - UI basics - sidebar, tables, forms, localization, searching
  - App Manager

<a class="btn" href="level-1/lesson-2"><span class="text">Open lesson #2</span></a> <span class="inline-flex items-center gap-2 align-middle ml-2 rounded-full border border-blue-200/25 bg-blue-400/5 px-2 py-1 text-xs font-medium leading-none text-blue-200/90"><i class="fas fa-video text-[0.7rem]"></i><span>Video available</span></span>

## Lesson #3: Business Apps Development - backend
  - Create custom CarRental app.
  - Overview of Model and RecordManager concepts.
  - Create models (Car, RentalHistory) and their RecordManagers.
  - Overview of 1:N relations.

<a class="btn" href="level-1/lesson-3"><span class="text">Open lesson #3</span></a> <span class="inline-flex items-center gap-2 align-middle ml-2 rounded-full border border-blue-200/25 bg-blue-400/5 px-2 py-1 text-xs font-medium leading-none text-blue-200/90"><i class="fas fa-video text-[0.7rem]"></i><span>Video available</span></span>

## Lesson #4: Business Apps Development - frontend
  - Overview of previous developments in custom CarRental app.
  - Overview of DescriptionAPI with practical example (describeTable).
  - Create controllers and views (Cars, RentalHistories).
  - Use built-in React UI components (`<app-table>`) in views.

<a class="btn" href="level-1/lesson-4"><span class="text">Open lesson #4</span></a> <span class="inline-flex items-center gap-2 align-middle ml-2 rounded-full border border-blue-200/25 bg-blue-400/5 px-2 py-1 text-xs font-medium leading-none text-blue-200/90"><i class="fas fa-video text-[0.7rem]"></i><span>Video available</span></span>

## Lesson #5: Business Apps Development - CarRental app
  - Overview of previous developments in custom CarRental app.
  - Recapitulation of backend programming.
  - Recapitulation of frontend programming.
  - Feedback from developers.

<a class="btn" href="level-1/lesson-5"><span class="text">Open lesson #5</span></a>

## Lesson #6: Custom App Development - IpInfoTest app
  - Development of IpInfoTest app

<a class="btn" href="level-1/lesson-6"><span class="text">Open lesson #6</span></a> <span class="inline-flex items-center gap-2 align-middle ml-2 rounded-full border border-blue-200/25 bg-blue-400/5 px-2 py-1 text-xs font-medium leading-none text-blue-200/90"><i class="fas fa-video text-[0.7rem]"></i><span>Video available</span></span>

## Lesson #7: Hubleto ReactUi basics
  - Overview of ReactUi library.
  - Using ReactUi in Twig.
  - Most commonly used properties of `<app-table>`.
  - Customizing table and form look & feel using DescriptionAPI.

<a class="btn" href="level-1/lesson-7"><span class="text">Open lesson #7</span></a>

## Lesson #8: Models, RecordManagers and Migrations, part 1
  - Overview of Model and RecordManager concepts.
  - Definition of relations.
  - Built-in record-manipulation API (record/save, record/delete, ...).
  - Practical examples for belongsTo and hasMany relations.
  - Customizing prepareReadQuery().

<a class="btn" href="level-1/lesson-8"><span class="text">Open lesson #8</span></a>

## Lesson #9: Models, RecordManagers and Migrations, part 2
  - Controlling relation loading for tables and forms.
  - Understanding table, form and lookup read pipelines.
  - Saving, validating and normalizing related records.
  - Defining indexes and unique constraints.
  - Evolving an existing database schema with additional migrations.

<a class="btn" href="level-1/lesson-9"><span class="text">Open lesson #9</span></a>

## Lesson #10: Description API
  - Overview of Description API, its purpose and basic principles.
  - Configuration options for tables (describeTable).
  - Configuration options for forms (describeForm).
  - Configuration options for inputs (describeInput).

<a class="btn" href="level-1/lesson-10"><span class="text">Open lesson #10</span></a>

## Lesson #11: Model callbacks
  - Overview of available callbacks.
  - Practical examples.

<a class="btn" href="level-1/lesson-11"><span class="text">Open lesson #11</span></a>

## Lesson #12: Controllers
  - Introduction to Hubleto controllers.
  - Practical examples.

<a class="btn" href="level-1/lesson-12"><span class="text">Open lesson #12</span></a>

## Lesson #13: Views
  - Introduction to Hubleto views.
  - Practical examples.

<a class="btn" href="level-1/lesson-13"><span class="text">Open lesson #13</span></a>

## Lesson #14: Integration with other apps, part 1
  - Calendar
  - Settings
  - Workflow
  - AI Assistant
  
<a class="btn" href="level-1/lesson-14"><span class="text">Open lesson #14</span></a>

## Lesson #15: Integration with other apps, part 2
  - Dashboards
  - Mail
  - Api
  - Tools
  
<a class="btn" href="level-1/lesson-15"><span class="text">Open lesson #15</span></a>

## Lesson #16: System-level apps
  - Desktop
  - Notifications
  - Developer tools
  - Audit Logs

<a class="btn" href="level-1/lesson-16"><span class="text">Open lesson #16</span></a>

## Lesson #17: Miscellaneous
  - Testing
  - Best practices for UI (friendly URLs, level 2 sidebar, app menu, breadcrumbs, ...)
  - Import / Export CSV
  - Localization
  - Sidebar badge

<a class="btn" href="level-1/lesson-17"><span class="text">Open lesson #17</span></a>

## Lesson #18: Finalization of Level 1 certification
  - What we have learned?
  - Feedback from developers.
  - How to get the Level 1 certificate.

<a class="btn" href="level-1/lesson-18"><span class="text">Open lesson #18</span></a>
