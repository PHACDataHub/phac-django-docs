## Conventions for local development environment

- We use virtualenv, not conda, pipenv, poetry, pdm, etc. 
- The root of the repo should be the one opened in vscode 
- The root of the repo should contain the pyproject.toml
- The root of the repo should contain the virtual environment
- The virtual environment should be called venv/

## Installing python

We're currently on python3.13 [Python Release Python 3.13.14 | Python.org](https://www.python.org/downloads/release/python-31314/)

pick the 64 bit installer

Using the installer, 
1. first screen: "Install Python 3.13.x (64-bit)"  
	- uncheck 'use admin rights' at the bottom
	- click customize installation
2. Second screen
	- make sure pip is checked
	- the others can be left unchecked
3. Third screen: Advanced Options
	- Can leave everything unchecked
	- **important:** Use a shorter path: `C:\Users\<MYUSERNAME>\Python313`
4. install virtualenv
	- `~/python313/python -m pip install virtualenv`

Global setup is now complete. Note that your basic `python` command won't be updated, you'll need to refer to the entire path. This may be annoying for data-science, but it's not a concern for development because we use virtualenvs for everything.

To create a virtualenv env in your project folder, you can create virtualenvs via `~/python313/python -m venv ./venv/`
Remember to completely delete any older virtualenvs 

If you need multiple versions of python (likely, because we can't upgrade all projects simultaneously), ideally this process did not mess with the old installation. Even if it did, it probably didn't break older virtual envs. But even if it breaks those, you can follow this same process to re-install the old python version and use that to re-create older venvs.

## installing and using postgres w/out sci-ops on windows:

1. download windows binaries: https://www.enterprisedb.com/download-postgresql-binaries (tested with v15)
2. extract to `C:\Users\{username}\AppData\Roaming\`
2. add path 
3. initdb -D -U postgres -E utf8 ~/pg/data/

This command is probably not necessary, but it will allow you to run "admin" commands without having to use `-U postgres` in front of everything.
`psql -U postgres -c "CREATE ROLE \"YOUR_UPPERCASE_USERNAME\" WITH LOGIN SUPERUSER"`


Here's what you'd run in a typical project to create a project-specific user and DB. In this case the username is `phacend` and the db name is `phac_dev`
1. psql -U postgres -c "CREATE ROLE phacend with login superuser"
2. psql -U postgres -c "ALTER ROLE phacend createdb"
3. createdb -U phacend phac_dev

the postgres can be started like this, it will shut down when you close the parent terminal window, so you may have to run this often, at least on every bootup. 
`pg_ctl -D ~/pg/data/ -l logfile start`


## Editor configuration

Django projects should be configured so that everyone opens VSCode in the root of the repository.

### Auto-formating python and jinja files

Projects should have the following autoformating config in their pyproject.toml files,

```toml
[tool.black]
line-length = 79
target-version = ['py310']
include = '\.pyi?$'
exclude = '''
(
  /(
    | \.git          # root of the project
    | venv
  )/
  # ignore auto-generated migrations, no one ever saves those manually
  | migrations/.*.py 
  | migrations/.*_initial.py 
  | migrations/.*_auto_.*.py 
)
'''


# this tells isort about our custom groups, django first-party, core-project and app. 
# If you have multiple apps, you can add groups
[tool.isort]
profile = "black"
line_length = 79
known_django = "django"
known_proj = "proj"
known_myapp = "myapp"
sections=[ "FUTURE", "STDLIB", "DJANGO", "THIRDPARTY", "PROJ", "MYAPP", "FIRSTPARTY", "LOCALFOLDER" ]
skip_glob=['*/migrations/*.py']


[tool.djlint]
indent=2
profile="jinja"
extension='jinja2'
preserve_blank_lines=true
```


Make sure vscode extensions are installed. If the project doesn't have a `.vscode/extensions.json` file, install the below. If you have other formating related extensions, you may want to disable them. 

- Python (microsoft)
- djLint
- Black formatter (microsoft)
- Better Jinja
- isort (microsoft)


### Autoformating with docker

Since vscode is not opened inside the docker container, you'll at least need to create a thin virtual environment that has all the format-related dependencies installed. If a project is configured for docker use, it should have a `requirements_formatting.txt` file.

Then, you can run the formating script from the root of the project. Run this in the root of the project, not the phacend/ directory

```bash
python -m venv venv
source venv/bin/activate # on *nix
./venv/Scripts/activate # on windows, venv/bin/activate on unix
pip install -r phacend/requirements_formatting.txt
```


## Testing VSCode configuration

1. Open a python file, move some imports out of order and save. If the imports don't automatically re-sort, the isort step is not working. 
2. Open a python file, add a bunch of newlines to a function. If the file isn't reformatted on save, there's a problem with the black step.
3. Open a jinja2 or jinja-html file. Add a bunch of lines and indent stuff the wrong way. If the file isn't reformatted on save, there's a problem with the djlint step.
