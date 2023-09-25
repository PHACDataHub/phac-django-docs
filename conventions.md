# Phac Django docs and conventions


## Modeling data

Getting the data-model right early on is the most important thing. Everything depends on the data-model, so if it's no good everything has to change.  

**Triple-check with partners, look for exceptions**

Partners will tell you the average case, e.g. "Projects have one project manager". Before you create a non-nullable `project_manager` foreign key on your `Project` model, interoggate them and see if there are exceptions. What about brand new projects, is it NULL then? Can a project have two managers if one is on vacation? 


**Choices vs lookup tables**

TLDR: Choice fields are fine for small, unchanging lists of simple labels. Use lookup tables when in doubt. 

Often times you'll want a dropdown on a model. There are 2 main ways to do this. Use a Char/Integer Field with a choices kwargs, or use a ForeignKey to a lookup table. A lookup table seems like more work, but it's better in many scenarios:

- You (or users) can add new choices without a code update
- You can't do many-to-many relationships with choices (at least not without building this yourself)
- You might have too many choices to fit into a python literal
- We want to attach additional data to the choices (django only supports adding a label)
- We want to iterate and nest data under the choices

The disadvantage to lookup tables is that they are heavier and require a bit more code to work with, e.g. N+1 issues pop up more often. You may have to sort your dropdown choices. You may have to create a `code` column on lookup table to use as a "dev-friendly" secondary unique key. 


## High level tips/conventions

**Stick to plain HTML/CSS as much as possible**

Django excels when it's used as a traditional web site in which 1 view = 1 page, possibly with a single form. Modals and bulk-editing should be used sparingly when the tasks absolutely requires it. 

**Use HTMX when appropriate** 

Good use-cases for HTMX
- toggle buttons (e.g. toggling a favorite and swapping out new button icon/text)
- "live" form validation errors for complex forms (only when it's necessary)
- cascading dropdowns (e.g. a city dropdown whose choices are dependent on an above country dropdown)
- most modals

**Avoid client side javascript**

- *Javascript is difficult to test*. If you write JS, make sure it's stuff that isn't modified often.
- Prefer HTMX over javascript
- JS is appropriate for focus management (sometimes necessary after HTMX swapping)
- Javascript is OK for bells-and-whistles that don't need testing
- Don't write business logic in JS
- Don't duplicate server logic (e.g. form validation) in JS
- jQuery datatables is a quick and dirty way to make smaller (<500 records) tables sortable and searchable


## Models

**Use our custom fields**

Plain django fields trigger migrations because their serialization includes kwargs like verbose_name, related_name, etc. Our custom fields will not trigger migraions unless DB-relevant kwargs are changed. (TODO: these fields are currently copied from one project to the next, we should integrate these into the helpers repo, but until then they can be copied from the catalog/itap repo.)

**Use `verbose_name` and maybe `help_text` for form labels**

Often we create multiple forms for the same model, repeating labels. ModelForms will automatically pick up verbose_name so it's a good idea to just put all the form labels there. 

**boolean fields should start with `is_*`, `should_*`, `has_*`**

e.g. Prefer `is_active` over `active`. 

**split up models.py** 

For small projects, a common way to split models is primary (e.g. `Project`, `Task`), lookups (e.g. `ProjectType`) and meta (e.g. `ProjectUserRole`). If a single model is too large, give it its own module. For larger projects, just create one module per class, it can be very annoying to have to jump around to find a model.


## HTTP

- Don't use DELETE/PUT/PATCH, only use GET/POST
- GET requests should be idempotent, (they should not modify the database)
- POST requests are ideally non-idempotent
    - POST requests are for create/edit/delete/trigger-action 
    - A search form should use GET params rather than POST params.
- In a standard non-htmx flow, successful form submissions should return 302 redirect, form errors should return 200.

## Views

Views should be as simple as possible. They should be thin wrappers around business logic.

- Avoid too much presentational logic in views. Don't assemble HTML strings or even CSS classes. That logic belongs in the template
- Avoid too much business logic in views. Queries is OK, but don't assemble complex data-structures. Complex logic belongs in queries, services, models or utilities. 
- A giant context dictionary is usually a code-smell (but sometimes necessary). Ask yourself if we need to pass smarter data-structures, or we need more jinja logic/helpers. 
- Passing the request object to a function is a code-smell


### class-based views (CBVs)

Use class-based views for everything. Why? Because they're more powerful than FBVs and standardizing helps with mixins.

This site makes navigating the class hierarchies very easy https://ccbv.co.uk/


- `View` is the base class of all CBVs, it has an important `dispatch` method
- `TemplateView` is very common and has an often overriden `get_context_data` method

- `FormView` is great when you have a single form
  - `CreateView` and `UpdateView` are even better wrappers around FormView, but they only apply if you want to create or modify existing models

Although many views are more easily written using FBVs, they won't play well with class-based mixins or decorators, so we should just stick with classes everywhere. 

***tripwire: MRO***

Most languages don't support multiple-inheritance, Python's flexibility comes at a cost. When a method calls `super()`, it's complicated to know which superclass' method will be called. This is defined by python's Method Resolution Order (MRO) algorithm. The algorithm will traverse the class hierarchy in a depth-first, left-to-right order. Generally this means that the left-most class will be called first, but python will ensure a class is never called before its descendants. 

If your mixins use super() and rely on other view base classes like `View` or `TemplateView` (e.g. because they override dispatch/get_context_data), they should override those respective classes, otherwise they'll be sensitive to left-right ordering.

### Function based views

Some projects have already standardized on FBVs. It may not be worth re-writing all these views. 

FBVs have a code re-use problem because you can't utilize inheritance. Never have one view function call another view function! Don't cheat by calling a function with the request and a bunch of arguments!  Instead, split behaviour into smaller functions, write similar views and accept the redundancy.

## Authorization

- When complex enough, use the rules framework
- If using CBVs, you can create reusable mixins that override `dispatch()` and raise `PermissionDenied`. If you trust views to use GET/POST consistently, you can use the request verb to conditionally check read vs. edit rules


## Low level tips 

- don't use super() with arguments, this is a python2 thing that is only used for hacky things in python3 (e.g. skipping a parent and calling a grandparent class)
- when passing arguments to `reverse()`, use `args=[...]` (lists) rather than `args=(..,)` (tuples), this will result in fewer bugs because (`(1)` is an integer, `(1,)` is a tuple)
- django's `@cached_property` method decorator is very useful for things you only wanted computed once, like unsaved models, forms, formsets, etc. It's much more readable than writing attributes onto classes within methods. 


## Splitting code into multiple apps? 

The recommended pattern is to have 2 apps:

1. One "generic core project" app that holds onto the user model, the login view and the base template.
2. Another "main" app with everything else

If there are models used by multiple different projects, e.g. PHACOrgs and Locations, it might make sense to put them into 3rd and 4th apps for ease of copy/pasting into other projects.

Django docs hint that you may want multiple apps but that can make it difficult to refactor and it turns migration dependencies into a _graph_, rather than a _line_. This graph should _never_ contain cycles or cirular dependency (e.g. foreign keys point both ways between apps). This is a sign it should have been a single app.

If your project is literally a combination of multiple projects (e.g. catalog and catalog-referencing indicators), it may also make sense to split those into 2 apps and have the one refer on the other.

