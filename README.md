# Github-actions

GitHub Actions have five important components:

1) Workflow — like a pipeline in Jenkins.

2) Events — triggers on which the workflow is run (e.g. Push, Pull Request, Scheduled).

3) Jobs — stages like Build, Test, Static Code Analysis.

4) Actions — reusable tasks inside jobs.

5) Runners — the platform / operating system that will execute the jobs.

Sample GitHub Flow:

.gitlab/workflow/.test_gha.yaml — GitHub Flow is to be created under Actions in GitHub, inside a repository.

After your repo is created, go to the Actions tab. There, you can create a workflow.

You can run the workflow manually as well. Also, you can disable it if you don’t want it triggered by any event.

Sampe workflow file structure:

name: demo_workflow        # “demo_workflow”: name of the workflow

on:                        # event on which the workflow is triggered.
  push:                    # triggers on push
    branches: [ "main" ]
  pull_request:            # triggers on pull request
    branches: [ "main" ]
  workflow_dispatch:       # allows manual run via the “Run workflow” button

jobs:                      # job is set of tasks that workflow executes
  build:                   # name of a job (unique)
    runs-on: ubuntu-latest # runner: Ubuntu environment

  steps:
      - uses: actions/checkout@v4
      - name: Run a one-line script
        run: echo "Hello, test change!"
      - name: Run a multi-line script
        run: |
          echo "Add other actions to build,"
          echo "test, and deploy your project."
