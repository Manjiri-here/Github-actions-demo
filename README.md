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

We define configuration variables in the settings of a repository, not inside the workflow’s YAML file. We use them in workflows by syntax ${{ vars.variableName }}.

To define a configuration variable (variableName)as : Go to Settings in the repository -> Then Secrets and variables → Actions -> Click New repository variable, define the name & value, and save it

Now you can run the workflow.

For values related only to a specific workflow, we use workflow variables. But for more common variables, we use configuration variables.

Context variable are preloaded set of information like who triggered the workflow, or repository name, or other metadata. These are available via the context from GitHub metadata.

Sometimes we have many conditions to run the workflow. For example, “if repo name is xyz”, “if job name is xyz”, or “if environment is xyz”, then run certain parts. That metadata (repo, workflow, or job) is accessed using context.

Example code using context variables:

```

name: context_variable_demo
on:
  workflow_dispatch

env:
  CLOUD: google

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Print repository and workflow info
        run: |
          echo "Repository name: ${{ github.repository }}"
          echo "Workflow name: ${{ github.workflow }}"
          echo "Triggered by: ${{ github.actor }}"
```

In the last three lines, we haven’t defined these variables anywhere. They are built-in and pulled from GitHub Actions contexts.

# Manual input and secrets:

>> Input under workflow_dispatch: You can also give manual input when triggering a manual workflow. We define this under inputs in the workflow_dispatch trigger. For syntax, refer to the GitHub Actions documentation or Vishal Bulbule’s repository. Then, when you manually trigger the workflow, you will get a window to enter those values. Inputs under workflow_dispatch allow manual workflows to ask for values. You define them in on.workflow_dispatch.inputs. 

>> Secrets: Now coming to secrets: you cannot pass secrets directly in the workflow file (hard-code them). Hence, you need to create secrets in GitHub. To create secrets, go to Settings → Secrets and variables → Actions. Then click New repository secret. Add the secret name and value and save it. 

>> In the workflow code, you use the secret the same way as configuration variables: ${{ secrets.secretName }}. When you run the step, the output will show stars (***) instead of the actual secret (masked). Use secrets with syntax ${{ secrets.SECRET_NAME }}. They are masked in logs.

Example of secrets and manual input are as below - (The workflow is this same repo : 'input_manual_trigger')

```
name: workflow_with_inputs_and_secrets
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Which environment to deploy to'
        required: true
        default: 'staging'
      run_tests:
        description: 'Run tests before deploy'
        required: false
        type: boolean
        default: false

jobs:
  job_one:
    runs-on: ubuntu-latest   # or self-hosted tag if using your own runner
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Greet with inputs
        run: echo "Deploying to ${{ github.event.inputs.environment }}"

      - name: Use a secret
        run: echo "Secret is ${{ secrets.test }}"  # will be masked
```
Output of manul input is as below (we get an option to mention environment)

<img width="1735" height="742" alt="Screenshot 2025-09-19 at 5 29 15 PM" src="https://github.com/user-attachments/assets/8505620f-997a-4a22-b7f1-ce51e56df972" />



# Runners:

Runners are machines that execute the jobs defined in GitHub Actions workflows. A workflow is a set of jobs; each job is a set of steps (commands). But where are these steps executed? On runners.

There are two types of runners:

1) GitHub-hosted runners: Virtual machines provided by GitHub. They come preconfigured (OS, tools installed, etc.). GitHub manages their maintenance.

2) Self-hosted runners: Machines you provide and manage (e.g. your EC2 instance). You install the runner software on them. You ensure network, security, uptime, etc.

To use a self-hosted runner:

Create the server in AWS or Azure as you normally do and then connect to them via SSH/terminal.
Ensure SSH (port 22) access if needed.
In your GitHub repository, go to Settings → Actions → Runners. In the Runners settings page, you’ll see commands to register the runner. Copy and execute those commands on the EC2 instance.

After registration, the runner appears in GitHub as active/self-hosted. In the workflow file use runs-on: self-hosted (or tag matching your runner) so GitHub knows to use your self-hosted runner.

When the EC2 instance (runner) is turned off or disconnected, the runner shows offline in GitHub. If you trigger a workflow then, jobs stay in queue. Also, not everyone has permission to register runners: depends on repository organization permissions. You can disable the self-hosted runner option in settings if needed.

Now configure value as 'self-hosted' for 'on' value in workflow as below-
```
runs-on: self-hosted

```

You need to implement the below commands to configure any server as runner (as mentioned above these commands are given in runner tab in Github). Also it is litening to jobs and running it. Once our workflow runs we can exit the runner. If after exiting runner if we go run the workflow again then it goes into queued state

<img width="1792" height="983" alt="Screenshot 2025-09-19 at 5 27 08 PM" src="https://github.com/user-attachments/assets/0f957625-05f8-4a5c-b28b-905f293c04a9" />


