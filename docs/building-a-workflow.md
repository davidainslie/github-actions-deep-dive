# Building a Workflow

We build a workflow in project [github-actions-deep-dive-lesson](https://github.com/davidainslie/github-actions-deep-dive-lesson).

The workflow is:
1. Check out the code to the Runner
2. Configure Python on Runner
3. Install the required libraries
4. Create a zipped bundle
5. Upload/Store the artifact

```yaml
name: Deploy my Lambda Function

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Install libraries
        run: |
          cd function
          python -m pip install --upgrade pip
          if [-f requirments.txt]; then
            pip install -r requirements.txt -t .;
          fi

      - name: Create zip bundle
        run: |
          cd function
          zip -r ../${{github.sha}}.zip .

      - name: Archive artifact
        uses: actions/upload-artifact@v2
        with:
          name: zipped-bundle
          path: ${{github.sha}}.zip
```

## Store/Publish Artifact

Examples to store artifacts;
- Sonatype Nexus
- JFrog Artifactory
- Cloud Storage buckets (S3)
- GitHub Packages

GitHub Packages supports:
- Container Registry
  - Store container images for `docker pull` commands and `Kubernetes deployments`.
- Ruby Gems
  - Store custom packages for Ruby code.
- Node modules
  - Store Node.js libraries for use with `npm`.
- Maven Registry
  - Store Maven depenedencies for Java apps.
- Gradle Registry
  - Store Gradle depenedencies for Java apps.
- NuGet
  - Store .NET packages for apps.

If your artifact is not one of these (such as Python), you can still store it as a `release asset`.

To publish we need to: build; authenticate; push.
The specific authentication method will differ depending on the type of package you are publishing.

Here is a Docker example:
```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{github.repository}}

jobs:
  build-and-push-image:
    ...
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Log into the Container registry
      uses: docker/login-action
      with:
        registry: ${{env.REGISTRY}}
        username: ${{github.actor}}
        password: ${{secrets.GITHUB_TOKEN}}

    - name: Build and push Docker image
      uses: docker/build-push-action
      with:
        context: .
        push: true
```

As we are dealing with a (Python) AWS Lambda Function, and AWS Lambda Pulls from S3... publishing to S3 is the way for us to go.