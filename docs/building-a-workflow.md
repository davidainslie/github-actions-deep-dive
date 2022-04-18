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

But before that, let's show how to create a GitHub release.

## GitHub Release

On our [lesson](https://github.com/davidainslie/github-actions-deep-dive-lesson) project, we'll do this on the branch `github-packages`.

The workflow is define in `deploy-pipeline.yaml` which has 2 jobs, `build` and `publish`:

```yaml
name: Deploy Lambda Function

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Install libraries
        run: pip install flake8

      - name: Lint with flake8
        run: |
            cd function
            flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
            flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

  build:
    runs-on: ubuntu-latest
    needs: lint

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
            if [ -f requirements.txt ]; then pip install -r requirements.txt -t .; fi

      - name: Zip bundle
        run: |
            cd function
            zip -r ../${{ github.sha }}.zip .

      - name: Archive artifact
        uses: actions/upload-artifact@v2
        with:
          name: zipped-bundle
          path: ${{ github.sha }}.zip

  publish:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Create release
        id: create-release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{secrets.github_token}} # This token is provided by Actions, you do not need to create your own token
        with:
          tag_name: ${{github.run_number}}
          release_name: Release from ${{github.run_number}}
          body: New release for ${{github.sha}} - Release notes on the documentation site.
          draft: false
          prerelease: false

      - name: Download artifact
        uses: actions/download-artifact@v2
        with:
          name: zipped-bundle

      - name: Upload release asset
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{secrets.github_token}}
        with:
          upload_url: ${{steps.create-release.outputs.upload_url}}
          asset_path: ./${{github.sha}}.zip
          asset_name: source-code-with-libraries.zip
          asset_content_type: application/zip
```

## Uploading to AWS

Configuring AWS Access:

#### Option 1 - Do it yourself

Add steps to the job to download and install the AWS CLI and all its dependencies, configure them on the runner, and set up credentials e.g.

```yaml
steps:
  - name: Download AWS CLI
    run: curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

  - name: Unzip AWS CLI
    run: unzip awscliv2.zip

  - name: Install AWS CLI
    run: sudo ./aws/install

  - name: Configure AWS CLI
    run: |
      export AWS_ACCESS_KEY_ID=${{secrets.AWS_ACCESS_KEY_ID}}
      export AWS_SECRET_ACCESS_KEY=${{secrets.AWS_SECRET_ACCESS_KEY}}
      export AWS_DEFAULT_REGION=${{env.AWS_REGION}}
```

#### Option 2 - Community action (recommended)

```yaml
steps:
  - name: Configure AWS CLI
    uses: aws-actions/configure-aws-credentials@v1
    with:
      aws-access-key-id: ${{secrets.AWS_ACCESS_KEY_ID}}
      aws-secret-access-key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
      aws-region: ${{env.AWS_REGION}}
```

Note - GitHub secrets are `encrypted` environment variables.

Let's create a S3 bucket for the artifact - For our purposes I created bucket:

| [my-bucket-for-workflow-demo](https://s3.console.aws.amazon.com/s3/buckets/my-bucket-for-workflow-demo?region=us-east-1) | US East (N. Virginia) us-east-1 |
| ------------------------------------------------------------ | ------------------------------- |

We create an AWS `user` to get access keys that we can use in the GitHub workflow.

Within IAM:
- Create a user - I've named this user `github-actions-user`.
- Just include `programmatic access` (which will generate keys).
- `Attach existing policies directly`:
  - `AWSLambda_FullAccess`
  - `AmazonS3FullAccess`

(Save the keys somewhere safely, such as hidden directory e.g. `.credentials`)

In GitHub, navigate (via tabs) to `Settings` > `Secrets` > `Actions` > `New repository secret`.

Add secret:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

## Deploying the Lambda Function

First create an `IAM role` that the lambda function can assume. So, navigate to:

`IAM > Roles > Create Role > AWS service > Lambda`

We don't need any policies attached - it just needs to exist. And we'll give it the name `my-lambda-role`.
The default selected trusted entities will be:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Principal": {
                "Service": [
                    "lambda.amazonaws.com"
                ]
            }
        }
    ]
}
```

And now we create the Lambda Function. Navigate to `Lambda > Create Function` providing the following:
- Function name: `my-function`
- Runtime: `Python 3.8`
- Use and existing role: `my-lambda-role`

We finish off by adding a `deploy job` to our workflow.

Once deployed, we can test the function with a `test event` such as:
```json
{
  "input": "Hello"
}
```