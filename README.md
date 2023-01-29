# Common CI
Commonly used CICD templates that can co-exist with versioning to be referenced
in other projects. This makes it easier to manage them when changes are made to
avoid making repetitive changes across projects.

## Usage
Reference a template from this repository similar to the examples below. Note
that the `on` logic is still required for each `pipeline` run context; however, the
jobs themselves are referenced.

Each pipeline assumes that if it is being run then the conditions for its execution
have been met.

## General info
Workflows will differ depending on the stack that's being used and the project
goal.

### Things to note
* Some projects require secrets for the pipelines
* `Makefile` presence is enforced, this is to create a "source of truth" for the
project both in CICD and for local development

## Example setups
A few examples are provided to illustrate the referencing.

### Golang package
A Golang package is likely going to need:
1. CI job on every commit (fmt, lint, test)
1. Binary release build on tag push to master/main

#### Golang package CI and deployment
This is an example setup for 2 pipelines, `CI` and `Release`.

**Note:** To utilize `golang-release.yml` you must have a Personal Access Token
created at the `repo` level with write permissions to releases. This is because
the `GITHUB_TOKEN` environment variable does not allow you to write from a called
reusable workflow.

```yml
# ci.yml
name: CI

on:
  pull_request:
  push:
    branches: ["master", "main"]
    paths-ignore: ["docs/**"]

jobs:

  ci:
    uses: mattdood/common-ci/.github/workflows/golang-ci.yml@v0.0.6

# release.yml
name: Golang-Release

on:

  push:
    branches: ["master", "main"]
    tags:
      - v*

jobs:

  # Technically both workflow_run and push will trigger
  # the pipeline, then it will falsely try to create a release
  # and fail. To avoid this, start with a ref on tags and
  # successful completion of CI
  release:
    permissions:
      contents: write
    uses: mattdood/common-ci/.github/workflows/golang-release.yml@v0.0.6
    secrets:
      GH_TOKEN: ${{ secrets.GH_PAT_TOKEN }}
    if: ${{ startsWith(github.ref, 'refs/tags/v')}}

```

### Python packages
A great example of how a Python package might be set up would be:
1. CI jobs to run on every commit (lint/test)
1. PyPI Test deployment on every push to master/main
    1. Requires CI to run
1. PyPI Prod deployment on every tag push to master/main
    1. Requires CI to run

#### PyPI package CI and deployment
The above illustrates a parent-child relationship between the PyPI deployment
steps and the CI, ensuring lint/test are run for everything. Below is an example
of a general `CI` pipeline being reused in deployment pipelines that check its
completion status.

```yml
# ci.yml
name: CI

on:
  pull_request:
  push:
    branches: ["master", "main"]
    paths-ignore: ["docs/**"]

jobs:

  ci:
    uses: mattdood/common-ci/.github/workflows/python-ci.yml@v0.0.0

# pypi-test.yml
name: PyPI-Test

on:

  workflow_run:
    workflows: [CI]
    types: [completed]

jobs:

  pypi-test:
    uses: mattdood/workflows/.github/workflows/pypi-test.yml@v0.0.0
    secrets:
      TEST_PYPI_API_TOKEN: ${{ secrets.TEST_PYPI_API_TOKEN }}
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

# pypi-prod.yml
name: PyPI-Prod

on:

  push
    tags:
      - '*'

jobs:

  pypi-prod:
    uses: mattdood/workflows.github/workflows/pypi-prod.yml@v0.0.0
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      PYPI_API_TOKEN: ${{ secrets.PYPI_API_TOKEN }}
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

