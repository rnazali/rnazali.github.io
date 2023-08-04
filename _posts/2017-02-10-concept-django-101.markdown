---
layout: post
title:  "Quick note on picking Django as your web framework"
date:   2017-02-10 17:19:27 +0700
published: false
---

## Why you might want to use Django
- Lot of commonly used built-in features ready to be used:
    - ORM-based model, and database migration
    - Admin panel
    - Unit test
    - Error reporting via email
    - Middleware customization (with handy built-in like redirect and 404)
    - Caching (with modular cache backend such as `memchached` or `redis`)
- Keeps you at the portion that really matters to your project instead of handling boilerplate
- Really fast time cycle from idea to a real product. It's motto:  `The web framework for perfectionists with deadlines.`
- Easily scalable on a containerized architecture
- Open source


## Design/Structure of the framework
- Inside a Django project, you can have many submodules (or application)
  - Submodules help maintain the modularity of the application.
  - Each submodule should only do one thing in common.
  - Naming the submodule should be straightforward and only use 1 to 2 words. 
  - If you need more word, you probably need to extract it to a different submodules.
  - For example, your restaurant app has `Menu` and `Order`, it should be separated to 2 submodules.
- Instead of the highly popular Model View Controller (MVC) pattern, Django uses `Model View Template (MVT)`.
  - Both `Model` can be treated as identical.
  - The Django's `View` is identical to `MVC's Controller`. It handles application logic.
  - About Django's `Template`:
    - When overly simplified, you can think of it like a React pure function/component.
    - A View will pass required values when rendering a template.
    - Template will render an HTML page according to how it is programmed to behave.
    - One template can be called/rendered by multiple views (reusability pattern).
- About application setting/configuration:
  - You can configure the entire Django site with a setting file. This cover things such as:
    - What database you are using (sqlite, mysql, postgres, etc) and its address
    - What host you are using for production mode
    - What middleware you are using, and so on.
  - You can have as many settings file as you need, and as modular as you like.
  - Configuration can also be nested. For example:
    - A folder with several configuration files as follows:
      - base_settings.py
      - dev_settings.py
      - prod_settings.py
    - The `base_settings.py` can hold any "common" configuration.
    - While `dev` and `prod` can import the setting from `base` and override as it may need to be.
  - In the end, Django will need exactly one setting to be used.

## How dependencies are managed
- By convention, all required packages/dependencies are stored inside `requirements.txt` that lives on the root project.
- This convention seems to also be applied to the whole python project.
- **Always** pinpoint a package's version inside `requirements.txt`, or you will have dependency hell time bomb.
- `pip` is the default package manager for python. It doesn't really have locking mechanism (as I'm aware of).
- One of the available alternatives are `pipenv`, which act quite like npm or yarn that really locks the dependencies version.
- To add a package with pip, simply run `pip install`
- To list all the currently installed dependency, run `pip freeze`
- To dump all the currently installed dependency to a text, use this usefull command `pip freeze > requirements.txt`

## How database migration is handled

### Assign the state with `makemigrations`
When running `python3 manage.py makemigrations`, django will:
- Compare all the **migration files** so far with all the currently available Models
- If Django detects a mismatch between the two, it assumes that something new is being addded/changed/deleted
- Django will then create a new **migration files** according to the currently available models

**Important**: 
- At the moment, nothing has been changed inside the database yet. 
- If there is no change between the previous state (as a collection of migration files) and currently declared model, 
then Django will not create any new migration file

Alternatively, you can add `--dry-run` flag as suffix to the command to see the direct impact of the command 
without ended up creating new migration files.

### Apply the state to the database with `migrate`

When running `python3 manage.py migrate`, django will:
- Collect all the migration files, and compare it with the current `migrations` table inside the database
- For any files that have not yet recorded in the table, Django will run it. 
(This is the exact point where database starts being modified/altered).
- After successful alter operation, Django will "take a note" on this by adding a record to the `migrations` table 
along with the name of the migration files. 

### Useful migration commands

Alternatively, you can also play more with migration with these commands:
- `python3 manage.py migrate menu` will only migrate the `menu` submodule to the newest state
- `python3 manage.py migrate menu 0005` will only migrate the `menu` submodule back/forth to state number `0005`
  - You can crosscheck this by reading the migration file inside `menu` submodule that starts with `0005_xxx.py`
- `python3 manage.py migrate menu zeru` will revert back `menu` submodule to state zero, basically means to delete all the tables.
