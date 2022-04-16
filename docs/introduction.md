# Introduction

## Workflow

A `workflow` is a collection of jobs that run based on a specific trigger. Conceptually, a `workflow` is a `CI pipeline`.
Workflows are defined in `YAML` files.

## Runner

A `runner` is a compute machine where workflows are executed. These can be GitHub managed runners or custom runners.

## Job

A `job` is a set of `steps` that execute in a single runner workspace.

# Step

A `step` is the lowest unit of functionality for GitHub Actions.
It can be a command, a script, a JavaScript file, a Dockerfile or a community action.

## Example

```yaml
# Workflows have a name
name: Build Application Code

# How is this workflow triggered?
on: [push]

jobs:
  # First job
  build:
    # What OS for the runner?
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v2

      - name: Install libraries
        run: pip install -r requirements.txt -t .

  # Second job which depends on first job completing successfully
  test:
    runs-on: ubuntu-latest
    needs: build

    steps:
      ...
```

#### Gitflow:

> The idea is to use branches to manage deployment e.g. the `main` branch is deployed to `prod` and `dev` branch is deployed to the `dev` environment.
> So each branch would have its own CI, and so constantly triggered and deployed to an associated environment.

## Community Actions

What are [Community Actions](https://github.com/marketplace)?

They are prepackaged steps for GHA. Community actions provide reusable templates for common steps.

## Runners

What is a `Runner`?
- A VM where jobs defined in a workflow are run
- Can be Linux, Mac or Windows
- Can run in a cloud, data centre or on a local workstation

We can use a `custom runner`, but be warned - You are responsible for security patching and updates on custom runners.