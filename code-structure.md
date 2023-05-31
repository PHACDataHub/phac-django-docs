This is what a typical project structure should look like,



root/
- pyproject.toml
- docs/
- manage.py
- tests/
    - conftest.py
    - tests_for_domain1/
        - conftest.py
- proj
    - settings.py
    - models.py 
    - urls.py
    - views.py
    - utils.py
    - model_utils.py
    - fields.py
    - templates/
        - base.jinja2
        - login.jinja2
    - services.py
    - queries.py
    - jinja_helpers.py
- app1
    - models/
        - subdomain1.py
        - subdomain2.py
    - views/
    - templates/
    - urls.py
    - queries.py
    - services.py
    - export.py


## The generic "proj" directory    

First, you'll see we have a `proj/` directory with its own models and view. This is where all the "global" project stuff goes. 

- The models will usually only contain a User model. Maybe you can throw in some other generic models like PHACOrg or locations. 
    - ALWAYS create your own user class, don't rely on django creating it for you  
- The views and urls will just contain a login, logout and root-redirect view. 
- The utils are going to be basic functions missing from python that any project might use, like `flatten()` or `group_by()`
- `fields.py` will contain our custom fields, described in the conventions doc. 
    - we should integrate these into the helpers repo, but until then they can be copied from the catalog/itap repo.
- model_utils will contain tools that any model might use. Like `@add_to_admin`, `@add_history` and `@add_history_and_author` decorators.

## The "app1" directory

This is where the meat of the project goes. All business logic should go here. Ideally, a django project only has one "app" like this (see conventions docs on app-splitting considerations).

**queries.py** (optional)

The queries module is where complex queries or data in complex format can be computed. 

When you need data that isn't a trivial query, you should put it here. It is especially useful when you're traversing multiple layers of a hierarchy. It can be tempting to just write loops in your views or templates but you can do a lot of pre-processing and smart, re-usable filtering in this module. 

It's OK for models to expose class methods that return querysets or lists, but when something spans multiple models it's better to use this separate module. 

**services.py**

The services module is where complex business logic can be computed. Unlike queries, these functions have side-effects. Nearly every project should have service functions for creating particular types of users. If you have generic utilities that modify records, they can also go here. 

**export.py**

Usually a project will have utilities for dumping lists or querysets into excel files. We have re-usable tools here, they should be added to the helpers repo but until then you can copy them from the catalog repo. 

