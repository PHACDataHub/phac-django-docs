# Git workflow

**Always merge PRs via squash-and-merge**

## Writing good pull requests 

**Keep it small! One single change**

Web applications are not like data-crunching scripts. You will spend more time reading code than writing it. The same goes for reviewing code. 

Reviewing code is hard. The difficulty increases super-linearly with the number of changed lines. It's easier to review five independent 100-line PRs than one multi-feature 500-line PR.

Ideally a pull-request changes a single thing and doesn't involve many changed lines. Try not to combine multiple tickets into one pull-request. If you want to refactor something and it's not completely related to a feature, do it in a separate pull-request. Don't be afraid to create many pull-requests for a single (or non-existing) ticket, efficient reviews are more important than project-management stat accuracy. 

If you want to include multiple changes, e.g. because your feature depends on a refactor, you can try and split out those different streams of work into different commits and direct reviewers to look at specific commits.

If you didn't split out your commits, you can still write a good pull-request by explaining the different streams of work and why they are combined.


**Run tests first!**

Most projects will have CI run tests for you. If you want to ask for review before CI has completed its checks, at least run them yourself locally before asking for review.

**Self-review**

Review your own PR first, you'll be surprised what you find! **Remove any print/debug statements, typos and commented out code**. If the PR contains weird code that needed code-comments, you can add a comment to draw attention. If your code is introducing new ideas, you can also write explanation in your PR's comments. 

Don't be afraid to leave yourself notes. Sometimes your PR is breaking new ground with a technique/pattern that someone (usually you!) may want to copy in the future. 


**Write a good description**

- Explain what your PR does and what problem it solves
    - Even if there's a linked azure ticket, many devs use off-network machines and can't quickly follow those links
- Give detailed instructions on how to test the changes locally.
- Add **screenshots** of your changes. Before/After screenshots are especially useful.


**Write tests**

- If you're adding new funcitonality, at least cover your new code
- Consider improving existing tests. If you're changing app logic and tests _didn't_ break, perhaps the existing tests weren't precise enough? 


**Remove dead code**

If your feature is replacing an older one, don't leave the old feature around nor comment it out. Just delete it and document what you're doing in the PR. Don't forget to also delete translations. 

**Writing a good PR review**

- Don't be afraid to ask questions. Pull-requests are a searchable source of documentation for developers. 
- Be clear about what you want changed, optionally changed or just want to discuss 
- Please test the PR locally
    - you can avoid creating a redundant local branch by using `git checkout origin/<branch-name>`, that way you don't have to delete a local branch afterwards
- If you're not sure about something, pull a 3rd or 4th reviewer in


## Branch Naming Convention (azure integrated projects)

If the project configured with azure dev-ops, we use a branch naming convention

Your branch name should follow a convention similar to the following:
\####-AA-description

Branch naming breakdown:

- \#### = 4 digits that is associated to the Azure DevOps work item ID that your branch is aiming to complete
- AA = your initials to keep track of each person's branch that might be also working on the same work item ID
- description = addition description/information related to the work item


If your branch isn't named correctly, e.g. because a ticket was created after the branch, you can still trigger a link to a ticket by including the ticket number in this format "Fixed AB####" in your PR's text body. 


## Conventional Commits (release-please integrated repos)

[Conventional commits](https://www.conventionalcommits.org/en/v1.0.0/) is a set of commit naming conventions that help standardized commits. In turn, this enables automatic incrementation, and documentation, of software releases (e.g., 0.6.1 to 0.7.0).

They look like this: "<span style="color: pink"><i>type</span><span style="color: lightblue">(optional scope)</span>: <span style="color: #F8DE7E">description</span></i>", where the optional scope is a noun describing a section of the codebase (e.g., api, template, deps)

```
    fix: patched bug causing add-to-cart to fail
    feat(api): added new unit test for payment gateway api //optional scope added
    build(deps)!: upgraded node version to 18.10.0 //! means a breaking change has been introduced
```

[List of types](https://github.com/pvdlg/conventional-commit-types#commit-types) for creating conventional commits:

- <b>build</b> ➡ pertains to packages
- <b>chore</b> ➡ updating grunt tasks where no production code changes (e.g., tool changes, configuration changes, and changes to things that do not actually go into production at all)
- <b>ci</b>
- <b>docs</b>
- <b>feat</b> ➡ a commit adds a new feature to your application or library
- <b>fix</b> ➡ a commit represents a bug fix for your application
- <b>perf</b> ➡ a commit that improves performance
- <b>refactor</b>
- <b>revert</b>
- <b>style</b>
- <b>test</b>

Conventional commits may seem silly but they enable automatic versioning and changelog generation. Once a project is well past the prototyping stage, this is essential. 

Tip: conventional commits format are essential for what ends up in main, but since we always squash-and-merge, you can still write non-compliants commits in your branch and fix them up in the github merge UI before merging. 

