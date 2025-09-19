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
```
name: demo_workflow        # “demo_workflow”: name of the workflow

on:                        # event on which the workflow is triggered.

  push:                    # triggers on push
    branches: [ "main" ]

  pull_request:            # triggers on pull request
    branches: [ "main" ]

  workflow_dispatch:       # allows manual run via the “Run workflow” button

jobs:                      # job is set of tasks that workflow executes

  build:                     # name of a job (it is unique always)
    runs-on: ubuntu-latest    # runner: Ubuntu environment

  steps:                      # steps are individual tasks under jobs
      - uses: actions/checkout@v4
      - name: Run a one-line script   # name of the step
        run: echo "Hello, test change!"
      - name: Run a multi-line script
        run: |
          echo "Add other actions to build,"
          echo "test, and deploy your project."
```
In the above code, 'uses: actions/checkout@v4' : It uses a pre-built GitHub Action that clones your repository into the workspace on the runner machine. You always need your code to test, build, and do static code analysis. Hence, you almost every time use this line in the workflow.

# Variables in github actions: 

i) Environment variable-
env: This keyword is used to define variables in GitHub Actions. You can define variables scoped for:

1) Entire workflow

2) Contents of a job within a workflow

3) Specific step within a job

So, in short, env variables are defined within a workflow. They are in key-value pairs, where key is used to call the variable (with a dollar sign). For example, $cloud: google_cloud.

```
name: workflow_variable
on:
  workflow_dispatch
env:
  cloud: google_cloud    # Variable one (workflow level)

jobs:
  job_one:
    runs-on: ubuntu-latest
    environment:
      greet: hello        # Variable two (job level)
    steps:
      - name: "greet the viewer"
        run: echo "$greet $first_name, welcome to $cloud!"
        env:
          first_name: Alex  # Variable three (step level)
```

Why this matters & supported behaviour (based on official docs):

Environment variables (env) can be defined at workflow, job, or step level. The scope of the variable depends on where it is defined: workflow-level variables are visible everywhere, job-level only inside that job, step-level only inside that one step. 
Default and custom variables, plus secrets, are supported.

ii) Configuration variable-
Now, if we want to declare a variable which can be used in multiple workflows in the repository, then we use configuration variables.



