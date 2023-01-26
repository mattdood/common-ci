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

    workflow_run:
        workflows: [CI]
        types: [completed]
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

