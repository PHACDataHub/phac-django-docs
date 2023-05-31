# Conventions for local development environment

- We use python 3.10
    - we'll have to upgrade eventually but we'll want to coordinate with other projects
- We use virtualenv, not conda, pipenv, poetry, pdm, etc. 
- The root of the repo should be the one opened in vscode 
- The root of the repo should contain the pyproject.toml
- The root of the repo should contain the virtual environment
- The virtual environment should be called venv/

## installing and using postgres w/out sci-ops on windows:

1. download windows binaries: https://www.enterprisedb.com/download-postgresql-binaries (tested with v15)
2. extract to `C:\Users\{username}\AppData\Roaming\`
2. add path 
3. initdb -D ~/pg/data/ -U postgres -E utf8

This command is probably not necessary, but it will allow you to run "admin" commands without having to use `-U postgres` in front of everything.
`psql -U postgres -c "CREATE ROLE \"YOUR_UPPERCASE_USERNAME\" WITH LOGIN SUPERUSER"`

If you ran 4, you don't need the `-U postgres` in front of everything. Of course, if something goes wrong, see if adding `-U postgres` can fix it

Project specific: 
1. psql -U postgres -c "CREATE ROLE phacend with login"
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
