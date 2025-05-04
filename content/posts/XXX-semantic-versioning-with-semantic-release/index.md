+++
date = '2025-05-05T08:00:00-05:00'
title = 'Versioning Software Artifacts The Easy Way'
slug = "semantic-versioning-with-semantic-release"
tags = ["ci/cd"]
+++

Versioning software has been a long debated topic. Should this be an automated process? Is it even worth automating, or is manually creating versions fine? If we do automate it, what is the naming convention? Should it be be based on time, the CI system build number, the git short sha, semantic versioning, or maybe something else?

I think we can all agree that yes, we should be versioning our software artifacts. We can also agree that ideally we should be using semantic versioning. 

### Semantic Versioning

Semantic versioning is ideal because it is easy for us humans to understand. I can compare two semantic versions and determine which one is newer, if the newer one will have a breaking change, or if the new version contains new features or just fixes to existing features.

```text
<major>.<minor>.<patch>
```

__Major__ - A breaking change has occurred to an existing feature. This could be due to a change in a contract or the functionality of a feature has changed.

__Minor__ - A new feature has been introduced.

__Patch__ - A fix to an existing feature has been triaged.

Now that we undertand the components of a semantic version, how can we automate the selection of bumping the major, minor, or patch for the next release? To be able to achieve this in an automatic fashion, we need to establish a contract that we will adhere to. This contract will allow us to determine the next version automatically. The contract that we will be adhering to is called Conventional Commits.

### Conventional Commits

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) is the specification, or contract, for writing human readable commits that can be interpretted by machines. This is the human to machine contract that we will be using to determine the next semantic version.

I strongly recommend that you review the official documentation for this specification. I will not be regurgitating it here in full, but essentially the specification states that all git commit messages should be in the format of:

```text
<type>(<scope>): <message or description>
```

In this format, there are three placeholders __type__, __scope__, and the __message or description__.

__type__

The overall type of change this git commit contains. This is what determines if the next semantic version should bump the major, minor, or patch version. There are a few types that we can use.
* feat
* fix
* docs
* ci
* chore

__scope__

The scope is the area of the codebase that is being modified in someway in this git commit. Remember, it is a development best practice to make small changes and integrate those changes often into the main branch.

This is an optional field, so it can be omitted. If you are working on a distributed system, it is likely not necessary. If this repository is for a monolithic application, it will probably be helpful to include the scope.

__Message or description__

The message or description that describes what changes were made. 

:warning: In our final solution, this message will be visible in a `CHANGELOG.md` file as well as on the releases page on the GitHub repository. Therefore, it is important to make this message meaningful for our future selves.

#### Examples

##### Example 1: Breaking change

The below is a breaking change to an existing feature and will result in bumping the __major__ version.

```shell
git commit -m 'feat!: This is a breaking change to an existing feature'
```

:warning: Note the exclamation mark after `feat`. This signifies there is a breaking change to an existing feature.

##### Example 2: New feature

The below is a new feature and will result in bumping the __minor__ version.

```shell
git commit -m 'feat: This is a new feature'
```

##### Example 3: Fix to an existing feature

The below is a fix to an existing feature and will result in bumping the __patch__ version.

```shell
git commit -m 'fix: This is a fix to an existing feature'
```

### Semantic Release

[Semantic Release](https://semantic-release.gitbook.io/semantic-release) is the tool that will interpret these conventional commits and determine what the next semantic version should be, build the artifacts, and push said artifacts to the artifact repository with this version.

#### Implementing A Semantic Release Workflow

#### Prerequisites

##### GitHub Repository

A private __GitHub repository__. It does not necessarily need to be private, but for this implementation we will be working under the idea that we are building and versioning software internal to our organization. This is likely the scenario for most of us.

##### GitHub Personal Access Token

A __GitHub Personal Access Token__ with repository access. We will need a PAT with repository access so that we have write access to our repository. 

We will need to be able to create new releases in github for our version, push the tag for our version, and optionally create a `CHANGELOG.MD` file as a nice bit of documentation.

##### GitHub Repository Collaborators

If you created the GitHub PAT under a service account user, be sure that this user is a collaborator with write access on this repository.

I know I've been burned a few times giving my PAT permissions, but not did not configurer permissions on the repository itself.

<!-- ![think](https://media4.giphy.com/media/v1.Y2lkPTc5MGI3NjExY3A5ZmN3cTBqY2RkaGl4bjVzN3ZnNGhzOGNld2IyYjN1bzZrcTJ6MyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/777Aby0ZetYE8/giphy.gif) -->

##### Configure Pull Request Commit Messages

Squash and merge.

Merge commit is PR Title.

#### Semantic Release Configuration

Before we can do anything with Semantic Release, we need to create a configuration file so that it will know how we want it to behave. 

For example, which branches it is allowed to be executed on and which plugins we want to use.

Create a file `release.config.js` at the root of your repository.

```js
const { execSync } = require('node:child_process');

module.exports = {
    branches: ["master", "main"],
    ci: false,
    plugins: [
        [
            "@semantic-release/commit-analyzer",
            {
              "preset": "conventionalcommits"
            }
          ],
          [
            "@semantic-release/release-notes-generator",
            {
              "preset": "conventionalcommits"
            }
          ],
          [
            "@semantic-release/github",
            {
              "successComment": "This ${issue.pull_request ? 'PR is included' : 'issue has been resolved'} in version ${nextRelease.version} :tada:",
              "labels": false,
              "releasedLabels": false
            }
          ],
          [
            "@semantic-release/changelog",
            {
              "changelogFile": "CHANGELOG.md",
              "changelogTitle": "# Changelog\n\nAll notable changes to this project will be documented in this file."
            }
          ],
          [
            "@semantic-release/git",
            {
              "assets": [
                "CHANGELOG.md"
              ],
              "message": "chore(release): version ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
            }
          ],
          {
            success(pluginConfig, context) {
                const { options: { repositoryUrl }, nextRelease: { version }, logger } = context;

                logger.info('Creating or updating major release tag!');
                const [major] = version.split(".");
                logger.info(`Pushing new version tag v${major} to git.`);
                execSync(`git tag --force v${major}`);
                execSync(`git push ${repositoryUrl} --force --tags`);
            }
          }
    ]
};
```

#### GitHub Actions Workflows

Now, to automate Semantic Release to be kicked off each time we merge a pull request to our main branch, we need to create two workflows.

The first workflow is technically optional, but I like to include it as it will prevent any issues with the conventional commit message. This workflow will validate that our pull request title is in the form of a conventional commit message. This workflow will fail the build if it is not.

Forgetting to write the commit messages in the form of a conventional commit can happen even to the most senior of engineers, so it is a nice precaution to have in place.

Create a file `./.github/workflows/pr-title.yaml` with the below content.

```yaml
name: 'Validate PR Title'

on:
  pull_request_target:
    types:
      - opened
      - edited
      - synchronize

jobs:
  main:
    name: Validate PR title
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
    steps:
      - uses: amannn/action-semantic-pull-request@v5.0.2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            fix
            feat
            docs
            ci
            chore
          requireScope: false
          subjectPattern: ^[A-Z].+$
          subjectPatternError: |
            The subject "{subject}" found in the pull request title "{title}"
            didn't match the configured pattern. Please ensure that the subject
            starts with an uppercase character.
          validateSingleCommit: false
```

#### Release Our First Version

### Conclusion
