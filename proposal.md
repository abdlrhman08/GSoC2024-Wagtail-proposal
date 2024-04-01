<style>
body {
    counter-reset: h1
}
h1 {
    counter-reset: h2;
    border-bottom: 0;
}
h2 {
    counter-reset: h3
}
h3 {
    counter-reset: h4
}
h2:not(.toc):before {
    content: counter(h2) ". ";
    counter-increment: h2
}
h3:before {
    counter-increment: h3;
    content: counter(h2) "." counter(h3) ". "
}
h4:before {
    counter-increment: h4;
    content: counter(h2) "." counter(h3) "." counter(h4) ". "
}
.toc:before {
    content: ""
}
</style>

<h1 align="center">Changing page's content type in Wagtail Admin</h1>
<p align="center">Google Summer of Code 2024 proposal for Wagtail CMS by Abdelrahman Hamada</p>

<h2 class="toc">Table of contents</h2>

1. [Abstract](#abs)
    1. [How is changing the page content type currently possible](#background)
    2. [Goals](#gls)
    3. [Benefits](#benefits)
2. [Implementation](#impl)
    1. [Overview](#ovr)
    2. [More Details](#det)
        1. [Page type change view](#view)
        2. [`HistoryView` of the new page](#hv)
        3. [Archiving feature](#af)
            1. [First approach](#fa)
            2. [Second approach](#sa)
        4. [Stream Fields](#sf)
        5. [Problems that may arise and their fix](#pf)
3. [Schedule and Milestones](#sm)
    1. [Community Bonding: Codebase understanding and more tickets](#cb)
    2. [First milestone: Page Menu Item, Basic functionality to change type](#fm)
        1. [`Change Page Type` Menu item and `ChoosePageType` view (1 week)](#cpt)
        2. [Base functionality for `ChangePageType` action (3 weeks)](#bf)
        3. [Tests for the base functionality of ChangePageType action (1 week)](#tbf)
    3. [Second milestone: Finishing `ChangePageType` action, Creating the view](#sms)
        1. [Finishing the `ChangePageType` action and start writing the view (2 weeks)](#fcpt)
        2. [Focusing on the view and making it look good (1 week)](#focus)
        3. [Finishing up the view, writing tests and documentation (1 week)](#fv)
    4. [Third milestone: Archiving feature (~2.5 weeks)](#tm)
        1. [Implementation of Page Archive (1.5 weeks)](#imp)
        2. [Writing tests (1 week)](#wt)
        3. [Two weeks extension if possible](#tw)
4. [About me](#abt)

<a name="abs"></a> 
## Abstract

<a name="background"></a> 
### How is changing the page content type currently possible

Currently the way to change the page type to another type is to simply create a new page with the wanted type, copy the original page\'s contents, move all children of the original page to the new page, a really hectic process.

Yet you will have two choices, either to delete the original page, or to keep it, change it\'s path and write your own checking code to prevent it from being served.

Another way you can edit the database manually by changing the `content_type_id` field to point to the wanted content type and create a new entry in wanted page type table, this is dangerous to do in production and any mistake can corrupt the database.

<a name="gls"></a> 
### Goals

The main goal is to provide a view and an action, which allows the editors to change their page\'s content type in a seamless way. The view would render having the contents of the original page contents and a preview of the contents of the new page beside each other. The fields of the new page which have similarity with the original page will be copied automatically to new page\'s fields. The original page\'s fields will be read-only, since we don\'t want to change them in this view, while we will alow the editor to change how the new page will still look like if he wants to. Once the editor is happy with the change he, can press submit and the action will execute.

The view will have a disclaimer notifying the user that the fields that were not copied automatically should be copied manually or they we will be lost.

Before the execution of the action, The editor will be redirected to a confirmation page to confirm the action. After confirmation the action will proceed with execution.

Another goal is to provide the editor the ability to archive pages. The original pages should automatically get archived after type change. Archiving is possible for any page not just the ones that we changed their content types. Archiving is possible by two ways, either moving the original page to a seperate tree or by creating a model that has a foreign key to the revision of the page before archiving.

<a name="benefits"></a> 
### Benefits

This features has been requested few times since 2018, first in [#4949](https://github.com/wagtail/wagtail/issues/4949) and later in [#7378](https://github.com/wagtail/wagtail/issues/7378). The problem of not being able to change the page type in Wagtail admin has also been discussed in [Stack-overflow](https://stackoverflow.com/questions/46736274/changing-page-type-via-wagtail-admin). Providing a user-friendly interface allowing the user to change page types seamlessly would allow the site to be future-proof for any page types needed, without the need of editing the database which could bring a lot of issues, or going through the whole process of creating a new page and copying the fields manually.

<a name="impl"></a> 
## Implementation

<a name="ovr"></a>
### Overview

Since this feature will be developed in a seperate package before packaging it with core, we will be using the hook functions `register_admin_urls` to register our new view, and we will be using `register_page_action_menu_item` to add the `Change Page Type` action to be available in the edit view of the pages.

We will implement a backend logic which will extract the fields of original page and return list containing the field\'s name, value and type. Then we will compare the field names and types with the new pages\' content type, then copy the fields that have the same name and type.

We will implement a `ChangePageType` action that is responsible for moving the data and the move the children of old page to the new page. It will be also responsible for archiving the old page for any emergency retrieval.

<a name="det"></a> 
### More Details

<a name="view"></a>
#### Page type change view

We will be adding a page action menu item, when pressed it will redirect to a view similar to the `add_subpage` view. It will list all the viable page models that the user can change to.

One the user chooses the type he wants, he is redirected to a new view. This view will have two forms. A form for the original page, which will be read-only and a form for the target page that is already filled with the data that could be transitioned. The user has the freedom to change any field.

Using the pages\' `TabbedInterface` here would be too much, since we don\'t need the promote panels. We will implement a custom `edit_handler` that will only have the content panels to be rendered. We will prevent editing of the title field since it must be the same in both pages.

I have implemented a simple helper function that will extract the fields of page and returns a list of tuples containing the field\'s name, type and it\'s value.

``` python
def page_fields_with_types(instance):

  if type(instance.specific) is not Page:
    instance = instance.specific
  else:
    raise Exception("Fields for root page can not be extracted")

    opts = instance._meta
    data = [
        (field.name, type(field), field.value_from_object(instance))
        for field in opts.local_concrete_fields[1:]
    ]

    return data
```

This helper function is just explaining the idea of the approach

If during instantiation of the new page type we set `path=(old pages path)` and delete the old page or move it to another location before saving the new page, we won\'t have to move the children of the old page one by one.

<a name="hv"></a>
#### `HistoryView` of the new page

While saving the new page we will set `log_action=None` in `Page.save()`, because we are not \"creating a new page\", we are just changing a content type.

We will register a new `log action` using the hook `register_log_actions`, naming it `wagtail.change_content_type`, this will be the first log entry of the history view.

<a name="af"></a>
#### Archiving feature

I have two approaches currently two implement the archiving feature.

<a name="fa"></a>
##### First approach

The first approach is creating a complete seperate tree to for archived pages. Any page that gets its type changes wil be automatically moved to the the archive tree. The `ChangePageType` action will take care of changing the path of the page.

The advantages of this approach is that this will prevent the archived page from being served without any checking code

The disadvantage here is, creating a new tree usually leads to the crash of Wagtail admin\'s index view. The crash happens due to in `wagtail.permission_policies.pages.PagePermissionPolicy.instances_with_direct_explore_permission` it is always assumed that there is only one tree and a query set of nodes of depth of 1 is returned, then in `PagePermissionPolicy.eplorable_root_instance` it tries to find a common ancestor for these pages, which is impossible if there exists another tree.

The patch for this would be to return the first root node if you are superuser, or return only the pages which are descendants of the root page and the user has valid permissions for them.

A viable patch would be:

``` python
def instances_with_direct_explore_permission(self, user):
    # Get all pages that the user has direct add/change/publish/lock permission on
    root = Page.get_first_root_node()
    if user.is_superuser:
        # superuser has implicit permission on the root node
        return [root]
    else:
        root_descendants = Page.get_descendants(root)
        codenames = self._get_permission_codenames(
            {"add", "change", "publish", "lock"}
        )
        return [
            perm.page
            for perm in self.get_cached_permissions_for_user(user)
            if perm.permission.codename in codenames
            and perm.page in root_descendants
        ]
```

This patch could also lead to performance issues, because `root_descendants` can be a giant queryset.

<a name="sa"></a>
##### Second approach

The second apporach is to create a `PageArchive` model. The model will have a `JSONField` which will store all field data of the original page, it\'s path, body, etc. When we change the content type of a page, the old page gets deleted, but the fields data will be stored in the JSONField of the `PageArchive` model.

When we try to retrieve a page, a new page gets created, we parse the `JSONField` and fill the fields with the data.

We will register `ListingView` for this model, and make it accessible with a menu item. We will not build a `ModelViewSet` because we will create `PageArchive` models only through page views.

The advantages of this approach is obviously Wagtail admin never breaks.

<a name="sf"></a>
### StreamFields

The helper function I wrote in [Page type change view](#page-type-change-view) section, when it finds a `StreamField`, it will return `StreamValue` instance. We can use this instance and extract the block types and compare them with block types in the new page to know which blocks to copy.

<a name="pf"></a>
### Problems that may arise and their fix

One of the problems that we will face, is what are we going to do with the models that were referencing the old page before changing it\'s type.

The approach to go with here is before deleting the old page, we can use `ReferenceIndex.get_reference_to()` to get the references pointing to the original page, then access these fields and point them to the new page type. If the field doesn\'t support the new page type, the relation will be dropped.

Another problem is that the revisions of the old page will get instantly deleted after deleting the page itself. If we decide to keep them even after page deletion, we can change the `object_id` and `content_type_id` they refer to be the new page, they will not be shown in `HistoryView` since we don\'t have `PageLogEntry` for them.

Also alias pages will be converted to ordinary pages.

<a name="sm"></a>
## Schedule and Milestones

Before starting to work on this project, I will do the following during the period of waiting for the accepted projects to be announed.

-   Familirize myself more with Django Treebeard and understand materialized path trees.
-   I want to understand more about Wagtail Admin\'s frontend, specially `Stimulus`, maybe it could be used in adding custom behaviour in the page type change view.
-   I will continue working on my current open tickets and work on other issues to be even more experienced with Wagtail\'s codebase.

My university exams will start on May 25th and will end on the 14th of June, during this period I will be quiet busy, I will still work for 1 or 2 hours a day, and if I don\'t code I will be researching more about the project. I will be happy if it is possible two have two weeks extension to finish the project.

On 16th of June I will be visiting my grand-parents during the first day of Eid-al-adha (An islamic holiday)

Beside those dates, I don\'t have anything else to do, even during holidays I stay at home, except if I am visiting my grand-parents on the first day. I am an introverted person, I don\'t go out a lot.

<a name="cb"></a>
### Community Bonding: Codebase understanding and more tickets

(From May 1 to May 25 \-- around 3.5 weeks)

During the community bonding period, I will

-   Hang out more with the community, get to know the community, help people on slack if I can.
-   Do more research on the Wagtail codebase, maybe I could improve my approaches implementing this project
-   I will work on any issues I could work on, report new issues I find, try to improve Wagtail overall

<a name="fm"></a>
### First milestone: Page Menu Item, Basic functionality to change type

I\'ll have exams at university so I\'ll start on June 14th and finish on July 12th \-- 5 weeks

I\'ll be implementing a page menu item, a view where the user selects the new page type and the base functionality of the `ChangePageType` action

<a name="cpt"></a>
#### `Change Page Type` Menu item and `ChoosePageType` view (1 week)

I\'ll start by creating the page menu item which redirects the user to the view where he can select the new type he wants. I\'ll be using the proper hooks to add it to the `PageActionMenu` and `PageListingButtons`

The menu item shouldn\'t take few days, so after I finish it I will start creating the view where the user selects the page type

<a name="bf"></a>
#### Base functionality for `ChangePageType` action (3 weeks)

After finishing the choose page type view, I will start implementing the base functionality of the `ChangePageType` action, I will start by writing my own prototypes at first, and I will resort to my mentors often to know their opinion about the approach. After we come to a solution I will start implementing.

<a name="tbf"></a>
#### Tests for the base functionility of `ChangePageType` action (1 week)

After I have implemented the base functionality for the `ChangePageType` action, I will be spending this week writing tests for it and documentation. After finishing the tests and documentation, I should have finished

-   `Change Page Type` menu item
-   The `Choose Page Type` view
-   Written the base functionality of the `ChangePageType` action

<a name="sms"></a>
### Second milestone: Finishing `ChangePageType` action, Creating the view

From the 12th of July to 9th of August \-- 4 weeks

During the second milestone period I will be finishing the `ChangePageType` action, I will also be implementing the view and the user-fiendly frontend interface where the user will be able to change the page type

<a name="fcpt"></a>
#### Finishing the `ChangePageType` action and start writing the view (2 weeks)

I believe the view and the action should be developed interchangeably. This will help me be more carefull while finishing the `ChangePageType` action, so in the first two weeks I will have

-   Wrote the basis of the view
-   Finished the `ChangePageType` action alongside continuing the view implementation
-   I should have written a template that has basic interactivity with the user

During the implementation of the view I will be resorting to my mentors often to know the opionion about the view

<a name="focus"></a>
#### Focusing on the view and making it look good (1 week)

In this week I will focus only on the view. I will implement the proper frontend interactivity, finish the template and make sure that the action and the view are working together correctly

<a name="fv"></a>
#### Finishing up the view, writing tests and documentation (1 week)

In this week I should finish the implementation of the view. I will also write tests for it and for the improved `ChangePageType` action.

So for I should have finished:

-   All the `Change Page Type` menu items and make it only available in the `Edit View` using the `is_shown()` method and in `PageListingButtons`
-   Implemented the `ChangePageType` action
-   Implemented and `Change Page Type` view
-   Tests and documentation for all the added functionality

Now changing pages content type should be doable in Wagtail admin, the only missing feature is archiving, which I will be implementing in the next few weeks

<a name="tm"></a>
### Third milestone: Archiving feature (\~2.5 weeks)

From the 9th of August till 25th

During the third milestone I\'ll be working solely on the archiving feature. and writing tests for it and documentation.

<a name="imp"></a>
#### Implementation of Page Archive (1.5 weeks)

Once me and the mentors settle on how to implement the archiving feature, I will implement it during this period.

So after one and a half I week, I should have finsihed:

-   The core functionality of the page archive should be completely finished, either through creating a new [revisions]{.title-ref} model like, as I mentioned in [Second approach](#second-approach) or a completely seperate tree.

<a name="wt"></a>
#### Writing tests (1 week)

In this week I will be writing the tests for archiving feature, and testing all the added features of the project intensively.

<a name="tw"></a>
#### Two weeks extension if possible

I would be really happy, if it is possible to get a 2 week extension to compensate the time that I was not working at full speed during my university exams.

The extension period will help me a lot in cleaning up, intensive testing and making sure everything is working correctly.

## The future of this project

<a name="abt"></a>
## About me

I am Abdelrahman A. S. Hamada. I am a third year computer engineering student at Ain Shams University in Egypt. I live in Cairo, Egypt, my timezone is UTC+02:00. I am currently 21 years old, I will be 22 next August. I\'ve been coding in Python for the last 1.5 years or more, I\'ve used Django in many personal projects. Currently, I have distributed systems project at my university, the backend is using Django and the frontend NextJS/React. I have had phases in my life where I have used Java, JavaScript and also C++ for a while.

I am fascinated by the web and I love backend development and backend architecture. I love the whole idea of computer networks.

I started contributing to Wagtail in early February. I currently have two open pull requests ([#11588](https://github.com/wagtail/wagtail/pull/11588) and [#11774](https://github.com/wagtail/wagtail/pull/11774)), both are part of Wagtail\'s 6.1 release planning and under review. I also have another pull request ([#11639](https://github.com/wagtail/wagtail/pull/11639)), but the approach I used wasn\'t the best approach to use, so I am trying to figure out a better one for this PR currently. I have also helped in fiew issues here and there. Before contributing I was trying to get a deep understanding of Wagtail\'s codebase.

I am planning to continue contributing to Wagtail even after the project is finished, I already feel like I belong to the community.

My email is `abdelrahmanhamada65@gmail.com`, also I am on Wagtail\'s Slack channels by my name, Abdelrahman Hamada
