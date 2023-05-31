# Django docs, tips and conventions

This a repo of docs for django projects. 


1. [conventions/advice for building and coding](conventions.md)
2. [Typical structure of a project](code-structure.md)
3. [Configuring local-development environments](local-dev.md)


## documentation TODOs:

- how to install python3.10 on windows
- how we use pytest, fixtures, etc. 
- how we do text and translation
- how we should use django-versionator and how to use it to create changelogs 
- how to treat "" vs null for char/text fields
- formsets vs htmx dynamic lists
- modal pattern
- solutions to N+1 queries
- scripts vs data-migrations
- authorization rules
- graphql? Can just link to [this](https://github.com/AlexCLeduc/graphene-django-promises-example/tree/master/proj/gql) for now
- convert this list into github issues

## Code TODOs:

- Create a template repo
- Extract some things into the helpers repo
    - custom model fields
    - export-utils
    - data-fetcher abstraction (clean up API first)
- Do something about versionator. Maybe fork it under PHAC org so others can add features. 
