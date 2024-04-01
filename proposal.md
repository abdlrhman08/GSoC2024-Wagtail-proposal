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

-- --
<br></br>

<h2 class="toc">Table of contents</h2>

- [Abstract](#abstract)
  - [How is changing the page content type currently possible](#how-is-changing-the-page-content-type-currently-possible)
  - [Goals](#goals)
  - [Benefits](#benefits)
- [Implementation](#implementation)
  - [Overview](#overview)
  - [More Details](#more-details)
    - [Page type change view](#page-type-change-view)
    - [`HistoryView` of the new page](#historyview-of-the-new-page)
    - [Archiving feature](#archiving-feature)
      - [First approach](#first-approach)
      - [Second approach](#second-approach)
  - [StreamFields](#streamfields)
  - [Problems that may arise and their fix](#problems-that-may-arise-and-their-fix)
- [Schedule and Milestones](#schedule-and-milestones)
  - [Community Bonding: Codebase understanding and more tickets](#community-bonding-codebase-understanding-and-more-tickets)
  - [First milestone: Page Menu Item, Basic functionality to change type](#first-milestone-page-menu-item-basic-functionality-to-change-type)
    - [`Change Page Type` Menu item and `ChoosePageType` view (1 week)](#change-page-type-menu-item-and-choosepagetype-view-1-week)
    - [Base functionality for `ChangePageType` action (3 weeks)](#base-functionality-for-changepagetype-action-3-weeks)
    - [Tests for the base functionality of `ChangePageType` action (1 week)](#tests-for-the-base-functionality-of-changepagetype-action-1-week)
  - [Second milestone: Finishing `ChangePageType` action, Creating the view](#second-milestone-finishing-changepagetype-action-creating-the-view)
    - [Finishing the `ChangePageType` action and start writing the view (2 weeks)](#finishing-the-changepagetype-action-and-start-writing-the-view-2-weeks)
    - [Focusing on the view and making it look good (1 week)](#focusing-on-the-view-and-making-it-look-good-1-week)
    - [Finishing up the view, writing tests and documentation (1 week)](#finishing-up-the-view-writing-tests-and-documentation-1-week)
  - [Third milestone: Archiving feature (~2.5 weeks)](#third-milestone-archiving-feature-25-weeks)
    - [Implementation of Page Archive (1.5 weeks)](#implementation-of-page-archive-15-weeks)
    - [Writing tests (1 week)](#writing-tests-1-week)
    - [Two weeks extension if possible](#two-weeks-extension-if-possible)
- [The future of this project (AI integration ?)](#the-future-of-this-project-ai-integration-)
- [About me](#about-me)


<div class="page"/>

## Abstract

### How is changing the page content type currently possible

Currently the way to change the page type to another type is to simply create a new page with the wanted type, copy the original page\'s contents, move all children of the original page to the new page, a really hectic process.

Yet you will have two choices, either to delete the original page, or to keep it, change it\'s path and write your own checking code to prevent it from being served.

Another way you can edit the database manually by changing the `content_type_id` field to point to the wanted content type and create a new entry in the wanted page type table, this is dangerous to do in production and any mistake can corrupt the database.

### Goals

The main goal is to provide a view and an action, which allows the editors to change their page\'s content type in a seamless way. The view would render having the contents of the original page and a preview of the contents of the new page beside each other. The fields of the new page which have similarity with the original page will be copied automatically to new page\'s fields. The original page\'s fields will be read-only, since we don\'t want to change them in this view, while we will alow the editor to change how the new page will still look like if he wants to. Once the editor is happy with the change he, can press submit and the action will execute.

The view will have a disclaimer notifying the user that the fields that were not copied automatically should be copied manually or they we will be lost.

Before the execution of the action, The editor will be redirected to a confirmation page to confirm the action. After confirmation the action will proceed with execution.

Another goal is to provide the editor the ability to archive pages. The original pages should automatically get archived after type change. Archiving is possible for any page not just the ones that we changed their content types. Archiving is possible by two ways, either moving the original page to a seperate tree or by creating a model that has a foreign key to the revision of the page before archiving.

### Benefits

This features has been requested few times since 2018, first in [#4949](https://github.com/wagtail/wagtail/issues/4949) and later in [#7378](https://github.com/wagtail/wagtail/issues/7378). The problem of not being able to change the page type in Wagtail admin has also been discussed in [Stack-overflow](https://stackoverflow.com/questions/46736274/changing-page-type-via-wagtail-admin). Providing a user-friendly interface allowing the user to change page types seamlessly would allow the site to be future-proof for any change in page types needed, without the need of editing the database which could bring a lot of issues, or going through the whole process of creating a new page and copying the fields manually.

<div class="page"/>

## Implementation

### Overview

Since this feature will be developed in a seperate package before packaging it with core, we will be using the hook functions `register_admin_urls` to register our new view, and we will be using `register_page_action_menu_item` to add the `Change Page Type` action to be available in the edit view of the pages.

We will implement a backend logic which will extract the fields of original page and return list containing the field\'s name, value and type. Then we will compare the field names and types with the new pages\' content type, then copy the fields that have the same name and type. Field type approximation is still a must, allowing only the same field types to be copied will cause the editor to do lot of the work himself.

We will implement a `ChangePageType` action that is responsible for moving the data and the move the children of old page to the new page. It will be also responsible for archiving the old page for any emergency retrieval.
 
### More Details

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

If during instantiation of the new page type we set `path=(old pages path)` and move the old page to another location before saving the new page, we won\'t have to move the children of the old page one by one. If we want to delete the old page we would have to set the path to an arbitrary location before deletion to prevent the deletion of child pages also.

#### `HistoryView` of the new page

While saving the new page we will set `log_action=None` in `Page.save()`, because we are not \"creating a new page\", we are just changing a content type.

We will register a new `log action` using the hook `register_log_actions`, naming it `wagtail.change_content_type`, this will be the first log entry of the history view.

#### Archiving feature

I have two approaches currently in mind for implementing the archiving feature.

##### First approach

The first approach is creating a complete seperate tree to for archived pages. Any page that gets its type changed will be automatically moved to the the archive tree. The `ChangePageType` action will take care of changing the path of the old page.

The advantages of this approach is that this will prevent the archived page from being served without any before-serve checking code.

The disadvantage here is, creating a new tree usually leads to the crash of Wagtail admin\'s index view. The crash happens due to in `wagtail.permission_policies.pages.PagePermissionPolicy.instances_with_direct_explore_permission` it is always assumed that there is only one tree and a query set of nodes of depth of one is returned, then in `PagePermissionPolicy.eplorable_root_instance` it tries to find a common ancestor for these pages, which is impossible if there exists another tree.

The patch for this would be to return the first root node if you are superuser, or return only the pages which are descendants of the root page and the user has valid permissions for them.

A viable patch would be:
<div class="page"/>

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

This patch could also lead to performance issues, because `root_descendants` could be a giant queryset.

##### Second approach

The second apporach is to create a `PageArchive` model. The model will have a `JSONField` which will store all field data of the original page, it\'s path, body, etc. When we change the content type of a page, the old page gets deleted, but the fields data will be stored in the JSONField of the `PageArchive` model.

When we try to retrieve a page, a new page gets created, we parse the `JSONField` and fill the fields with the data.

We will register `ListingView` for this model, and make it accessible with a menu item. We will not build a `ModelViewSet` because we will create `PageArchive` models only through page views.

The advantages of this approach is obviously Wagtail admin never breaks.

### StreamFields

The helper function I wrote in [Page type change view](#page-type-change-view) section, when it finds a `StreamField`, it will return `StreamValue` instance. We can use this instance and extract the block types and compare them with block types in the new page to know which blocks to copy.

### Problems that may arise and their fix

One of the problems that we will face, is what are we going to do with the models that were referencing the old page before changing it\'s type.

The approach to go with here is before deleting the old page, we can use `ReferenceIndex.get_reference_to()` to get the references pointing to the original page, then access these fields and point them to the new page type. If the field doesn\'t support the new page type, the relation will be dropped.

Another problem is that the revisions of the old page will get instantly deleted after deleting the page itself. If we decide to keep them even after page deletion, we can change the `object_id` and `content_type_id` they refer to be the new page, they will not be shown in `HistoryView` since we don\'t have `PageLogEntry` for them.

Alias pages will be converted to ordinary pages. Another approach is to create a field-to-field aliasing. This aliasing will only affect fields accordinly but not the page itself. This would introduce some kind of complexity, but not impossible to implement.

## Schedule and Milestones

Before starting to work on this project, I will do the following during the period of waiting for the accepted projects to be announed.

-   Familirize myself more with Django Treebeard and understand materialized path trees.
-   I want to understand more about Wagtail Admin\'s frontend, specially `Stimulus`, maybe it could be used in adding custom behaviour in the page type change view.
-   I will continue working on my current open tickets and work on other issues to be even more experienced with Wagtail\'s codebase.

My university exams will start on May 25th and will end on the 14th of June, during this period I will be quiet busy, I will still work for 1 or 2 hours a day, and if I don\'t code I will be researching more about the project. I will be happy if it is possible two have two weeks extension to finish the project.

On 16th of June I will be visiting my grand-parents during the first day of Eid-al-adha (An islamic holiday)

Beside those dates, I don\'t have anything else to do, even during holidays I stay at home, except if I am visiting my grand-parents on the first day. I am an introverted person, I don\'t go out a lot.

### Community Bonding: Codebase understanding and more tickets

(From May 1 to May 25 \-- around 3.5 weeks)

During the community bonding period, I will

-   Hang out more with the community, get to know the community, help people on slack if I can.
-   Do more research on the Wagtail codebase, maybe I could improve my approaches implementing this project
-   I will work on any issues I could work on, report new issues I find, try to improve Wagtail overall

### First milestone: Page Menu Item, Basic functionality to change type

I\'ll have exams at university so I\'ll start on June 14th and finish on July 12th \-- 5 weeks

I\'ll be implementing a page menu item, a view where the user selects the new page type and the base functionality of the `ChangePageType` action

#### `Change Page Type` Menu item and `ChoosePageType` view (1 week)

I\'ll start by creating the page menu item which redirects the user to the view where he can select the new type he wants. I\'ll be using the proper hooks to add it to the `PageActionMenu` and `PageListingButtons`

The menu item shouldn\'t take few days, so after I finish it I will start creating the view where the user selects the page type

#### Base functionality for `ChangePageType` action (3 weeks)

After finishing the choose page type view, I will start implementing the base functionality of the `ChangePageType` action, I will start by writing my own prototypes at first, and I will resort to my mentors often to know their opinion about the approach. After we come to a solution I will start implementing.

#### Tests for the base functionality of `ChangePageType` action (1 week)

After I have implemented the base functionality for the `ChangePageType` action, I will be spending this week writing tests for it and documentation. After finishing the tests and documentation, I should have finished

-   `Change Page Type` menu item
-   The `Choose Page Type` view
-   Written the base functionality of the `ChangePageType` action

### Second milestone: Finishing `ChangePageType` action, Creating the view

From the 12th of July to 9th of August \-- 4 weeks

During the second milestone period I will be finishing the `ChangePageType` action, I will also be implementing the view and the user-fiendly frontend interface where the user will be able to change the page type

#### Finishing the `ChangePageType` action and start writing the view (2 weeks)

I believe the view and the action should be developed interchangeably. This will help me be more carefull while finishing the `ChangePageType` action, so in the first two weeks I will have

-   Wrote the basis of the view
-   Finished the `ChangePageType` action alongside continuing the view implementation
-   I should have written a template that has basic interactivity with the user

During the implementation of the view I will be resorting to my mentors often to know the opionion about the view

#### Focusing on the view and making it look good (1 week)

In this week I will focus only on the view. I will implement the proper frontend interactivity, finish the template and make sure that the action and the view are working together correctly

#### Finishing up the view, writing tests and documentation (1 week)

In this week I should finish the implementation of the view. I will also write tests for it and for the improved `ChangePageType` action.

<div class="page"/>

So for I should have finished:
-   All the `Change Page Type` menu items and make it only available in the `Edit View` using the `is_shown()` method and in `PageListingButtons`
-   Implementation of `ChangePageType` action
-   Implementation of `Change Page Type` view
-   Tests and documentation for all the added functionality

Now changing pages content type should be doable in Wagtail admin, the only missing feature is archiving, which I will be implementing in the next few weeks

### Third milestone: Archiving feature (\~2.5 weeks)

From the 9th of August till 25th

During the third milestone I\'ll be working solely on the archiving feature. and writing tests for it and documentation.

#### Implementation of Page Archive (1.5 weeks)

Once me and the mentors settle on how to implement the archiving feature, I will implement it during this period.

So after one and a half I week, I should have finsihed:

-   The core functionality of the page archive should be completely finished, either through creating a new `Revision` like model, as I mentioned in [Second approach](#second-approach) or a completely seperate tree.

#### Writing tests (1 week)

In this week I will be writing the tests for archiving feature, and testing all the added features of the project intensively.

#### Two weeks extension if possible

I would be really happy, if it is possible to get a 2 week extension to compensate the time that I was not working at full speed during my university exams.

The extension period will help me a lot in cleaning up, intensive testing and making sure everything is working correctly.

<a name="the-future-of-this-project-ai-integration-"></a>
## The future of this project (AI integration ?)

I can see this is just a start of a great project. I plan to work on this still after the period of GSoC. This project would have a really great potential if integrated with AI. AI models would help the user during the type change action, field types wouldn't be always tied to the same field type in the new page. Changing content type maybe wouldn't even need much human intervention if the process is integrated with AI correctly.

I've been reading through Wagtail AI package and I really liked how it helps editors with their content. Integrating AI into this project would truly enhance its capabilities and effectiveness.

<div class="page"/>

## About me

I am Abdelrahman A. S. Hamada. I am a third year computer engineering student at Ain Shams University in Egypt. I live in Cairo, Egypt, my timezone is UTC+02:00. I am currently 21 years old, I will be 22 next August. I\'ve been coding in Python for the last 1.5 years or more, I\'ve used Django in many personal projects. Currently, I have distributed systems project at my university, the backend is using Django and the frontend NextJS/React. I have had phases in my life where I have used Java, JavaScript and also C++ for a while.

I am fascinated by the web and I love backend development and backend architecture. I love the whole idea of computer networks.

My contributions to Wagtail are:
- Add support for related fields in generic `IndexView.list_display` ([#11588](https://github.com/wagtail/wagtail/pull/11588))
- Add `ChooseParentView` to `PageListingViewSet` ([#11774](https://github.com/wagtail/wagtail/pull/11774) - Ongoing) (Sage M. helped me a lot in this one)

I started contributing to Wagtail in early February. I also have another pull request ([#11639](https://github.com/wagtail/wagtail/pull/11639)), but the approach I used wasn\'t the best approach to use, so I am trying to figure out a better one for this PR currently. I have also helped in fiew issues here and there. Before contributing I was trying to get a deep understanding of Wagtail\'s codebase.

I am planning to continue contributing to Wagtail even after the project is finished, I already feel like I belong to the community.

My email is `abdelrahmanhamada65@gmail.com`, also I am on Wagtail\'s Slack channels by my name, Abdelrahman Hamada
