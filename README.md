# Introduction
This README contains code for all pipelines used in CICD module of AWS Trainings.

Please go through presentation slides to get step-by-step instructions on how to configure CICD for a base application.
Slides can be found under:
* [Presentation](https://capgemini.sharepoint.com/:b:/s/TrainAWStrainers/EX179OtEeQBJi0-3BbY_RGIBwGWEkmae3TkHLsbfHp5E6g?e=NtkQYv)

# Preparation
1. Fork base repository to your account
   * https://github.com/Alegres/awstraining-cicd
2. Check if Destroy infrastructure workflow is visible under "Actions" tab

If not, then edit workflow file and just add some dummy commit data to make it visible for GitHub.

3. Clone fork to your computer
4. Replace **<<ACCOUNT_ID>>** in the whole project with your AWS Account ID
5. Go to ```aws-infrastructure/terraform/wrapper.properties``` file and set **UNIQUE_BUCKET_STRING** to some unique value that will be used as your Terraform state bucekt name
6. Go to Settings -> Secrets and variables and setup AWS credentials
   * BACKEND_EMEA_TEST_AWS_KEY
   * BACKEND_EMEA_TEST_AWS_SECRET  

# Hello world workflow
1. Go to GitHub -> Actions
2. Click on "New workflow" and then on "Set up a workflow yourself"
3. Set a proper workflow YAML file name
4. Write workflow code

```yaml
name: Hello world
run-name: Hello ${{ inputs.name }} run

on:
  workflow_dispatch:
    inputs:
      name:
        description: "Provide your name"
        required: true
        type: "string"
      last_name:
        description: "Provide your last name"
        required: true
        type: "string"
jobs:
  print_hello_world:
    runs-on: ubuntu-latest
    steps:
      - name: Print name
        run: |
          echo "Hello ${{ inputs.name}}! Welcome in this step!"
          
  print_last_name:
    runs-on: ubuntu-latest
    steps:
      - name: Print last name
        run: |
          echo "Your last name is ${{ inputs.last_name }}"-
      - id: count_characters
        name: Count characters
        run: |
          last_name="${{ inputs.last_name }}"
          length=${#last_name}
          echo "length=$length" >> $GITHUB_OUTPUT
    outputs:
      last_name_length: ${{ steps.count_characters.outputs.length }}

  print_last_name_length:
    runs-on: ubuntu-latest
    needs: print_last_name
    steps:
      - name: Display last name length in summary
        run: |
          echo "### Summary: Length of last name is ${{ needs.print_last_name.outputs.last_name_length }}" >> $GITHUB_STEP_SUMMARY
```

5. Commit changes
6. Go to GitHub -> Actions, select new workflow and execute it

# Custom composite action
1. Create new repository and place the custom composite action code in it

```yaml
name: 'Hello World'
description: 'Greet someone'
inputs:
  name:  # id of input
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  random-number:
    description: "Random number"
    value: ${{ steps.random-number-generator.outputs.random-number }}
runs:
  using: "composite"
  steps:
    - name: Set Greeting
      run: echo "Hello $INPUT_WHO_TO_GREET."
      shell: bash
      env:
        INPUT_WHO_TO_GREET: ${{ inputs.name }}

    - name: Random Number Generator
      id: random-number-generator
      run: echo "random-number=$(echo $RANDOM)" >> $GITHUB_OUTPUT
      shell: bash
```

2. Grant permissions for outside repositories to execute your custom action
3. Prepare new release and version tag for your action
4. In your main repository create a new "Greet workflow" and paste below code

Execution worfklow:
```yaml
name: Greet workflow
run-name: Greeting everyone

on:
  workflow_dispatch: {}
jobs:
  greet_everyone:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        name:
          - Damian
          - Piotr
          - Jan
          - Sergio
    steps:
      - name: Greet
        id: greet
        uses: Alegres/custom-hellow@v1
        with:
          name: ${{ matrix.name }}
      - name: Print action call output
        run: |
          echo "Output random number: ${{ steps.greet.outputs.random-number }}"
```

Please adjust path to point to your custom action repository.

5. Execute your "Greet workflow"

# Datacenter Map
Remember to adjust **<<ACCOUNT_ID>>** in workflow before execution.

```yaml
name: Datacenter map

on:
  workflow_call:
    inputs:
      hubEnv:
        required: true
        type: "string"
    # Map the workflow outputs to job outputs
    outputs:
      HUB:
        value: ${{ jobs.datacenterMap.outputs.HUB }}
      STAGE:
        value: ${{ jobs.datacenterMap.outputs.STAGE }}
      AWS_ACCOUNT:
        value: ${{ jobs.datacenterMap.outputs.AWS_ACCOUNT }}
      PROFILE:
        value: ${{ jobs.datacenterMap.outputs.PROFILE }}
      REGION:
        value: ${{ jobs.datacenterMap.outputs.REGION }}
      CLUSTER_NAME:
        value: ${{ jobs.datacenterMap.outputs.CLUSTER_NAME }}
      SERVICE_NAME:
        value: ${{ jobs.datacenterMap.outputs.SERVICE_NAME }}
      TASK_NAME:
        value: ${{ jobs.datacenterMap.outputs.TASK_NAME }}
jobs:
  datacenterMap:
    runs-on: ubuntu-latest
    outputs:
      HUB: ${{ env.HUB }}
      STAGE: ${{ env.STAGE }}
      AWS_ACCOUNT: ${{ env.AWS_ACCOUNT }}
      PROFILE: ${{ env.PROFILE }}
      REGION: ${{ env.REGION }}
      CLUSTER_NAME: ${{ env.CLUSTER_NAME }}
      SERVICE_NAME: ${{ env.SERVICE_NAME }}
      TASK_NAME: ${{ env.TASK_NAME }}
    steps:
      - uses: kanga333/variable-mapper@master
        with:
          key: "${{ inputs.hubEnv }}"
          map: |
            {
              "BACKEND_EMEA_TEST": {
                 "HUB": "EMEA",
                 "STAGE": "TEST",
                 "AWS_ACCOUNT": "<<ACCOUNT_ID>>",
                 "PROFILE": "backend-test",
                 "REGION": "eu-central-1",
                 "CLUSTER_NAME": "backend-ecs-test",
                 "SERVICE_NAME": "backend",
                 "TASK_NAME": "backend"
              },
              "BACKEND_US_TEST": {
                 "HUB": "US",
                 "STAGE": "TEST",
                 "AWS_ACCOUNT": "<<ACCOUNT_ID>>",
                 "PROFILE": "backend-test",
                 "REGION": "us-east-1",
                 "CLUSTER_NAME": "backend-ecs-test",
                 "SERVICE_NAME": "backend",
                 "TASK_NAME": "backend"
              }
            }
```

# Provision infrastructure with Terraform pipeline
1. Create a new workflow for provisioning the AWS infrastructure with Terraform

```yaml
name: Provision with Terraform
run-name: Provision with Terraform

on:
  workflow_dispatch:
    inputs:
      hubEnv:
        description: Select target hub and stage
        required: true
        type: choice
        options:
          - 'BACKEND_EMEA_TEST'


jobs:
  properties:
    uses: ./.github/workflows/datacenterMap.yml
    with:
      hubEnv: ${{ inputs.hubEnv }}
    secrets: inherit

  terraform:
    runs-on: ubuntu-latest
    needs: properties
    env:
      AWS_ACCOUNT: ${{ needs.properties.outputs.AWS_ACCOUNT }}
      AWS_PROFILE: ${{ needs.properties.outputs.PROFILE }}
      AWS_KEY:  ${{ secrets.BACKEND_EMEA_TEST_AWS_KEY }}
      AWS_SECRET: ${{ secrets.BACKEND_EMEA_TEST_AWS_SECRET }}
      AWS_REGION: ${{ needs.properties.outputs.REGION }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          # Disabling shallow clone is recommended for improving relevancy of reporting
          fetch-depth: 0

      - name: Install Terraform ${{ env.terraform_version }}
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.7
          terraform_wrapper: false

      - name: Configure AWS profile
        run: |
          mkdir -p ~/.aws
          echo "[${{ env.AWS_PROFILE }}]" > ~/.aws/credentials
          echo "aws_access_key_id = ${{ env.AWS_KEY }}" >> ~/.aws/credentials
          echo "aws_secret_access_key = ${{ env.AWS_SECRET }}" >> ~/.aws/credentials
      - name: Run Terraform apply
        run: |
          cd aws-infrastructure/terraform
          chmod +x setup_new_region.sh
          chmod +x w2.sh
          ./setup_new_region.sh w2.sh ${{ env.AWS_PROFILE }} ${{ env.AWS_REGION }} apply -auto-approve
```

# Multibranch pipeline
## Preparation
First, please set secrets (credentials) in AWS Secrets Manager:
```json
{
  "backend": {
    "security": {
      "users": [
        {
          "username": "userEMEATest",
          "password": "$2a$10$uKw9ORqCF.qA3p6woHCgmeGW0jFuU9AstYhl61Uw8RTQ5AaZCfuru",
          "roles": "USER"
        }
      ]
    }
  }
}
```

You also need to update task.json and replace **<<TODO: set ARN of secrets manager>>** with ARN of your secrets manager.

Then, please set **BACKEND_EMEA_TEST_SMOKETEST_BACKEND_PASSWORD** repository secret to "welt", as this is the password for the above test user, that will be used for smoke tests.

## Pipeline
1. Create the below pipeline

```yaml
name: Multibranch pipeline
run-name: Multibranch ${{ github.head_ref || github.ref_name }}

on:
  push:
    branches:
      - main
      - release/*
      - feature/*
      - bugfix/*
  workflow_dispatch:
    inputs:
      skipSmokeTests:
        description: Should skip smoke tests?
        type: boolean
        required: false
        default: false

jobs:
  properties:
    uses: ./.github/workflows/datacenterMap.yml
    with:
      hubEnv: BACKEND_EMEA_TEST
    secrets: inherit

  buildTestDeploy:
    runs-on: ubuntu-latest
    needs: properties
    env:
      AWS_ACCOUNT: ${{ needs.properties.outputs.AWS_ACCOUNT }}
      AWS_KEY:  ${{ secrets.BACKEND_EMEA_TEST_AWS_KEY }}
      AWS_SECRET: ${{ secrets.BACKEND_EMEA_TEST_AWS_SECRET }}
      REGION: ${{ needs.properties.outputs.REGION }}
      CLUSTER_NAME: ${{ needs.properties.outputs.CLUSTER_NAME }}
      SERVICE_NAME: ${{ needs.properties.outputs.SERVICE_NAME }}
      SKIP_SMOKE_TESTS: ${{ (github.event.inputs.skipSmokeTests != null && github.event.inputs.skipSmokeTests == 'true') || 'false' }}
      SMOKETEST_BACKEND_PASSWORD: ${{ secrets.BACKEND_EMEA_TEST_SMOKETEST_BACKEND_PASSWORD }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          # Disabling shallow clone is recommended for improving relevancy of reporting
          fetch-depth: 0

      - name: JDK 17 setup
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'

      - name: Maven setup
        uses: stCarolas/setup-maven@v4.5
        with:
          maven-version: 3.8.2

      - name: Cache local Maven repository
        uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-
      # Extract branch name from GITHUB_REF environment variable, which be default is in the format "refs/heads/<branch-name>"
      # e.g. extracts refs/heads/<branch-name> -> <branch-name>
      - name: Extract branch name
        if: ${{ env.SKIP_SONAR == 'false' && !github.base_ref }}
        run: echo "##[set-output name=branch;]$(echo ${GITHUB_REF#refs/heads/})"
        id: extract_branch
        
      - name: Display built branch in summary
        if: always()
        run: |
          echo "### Built branch ${{ steps.extract_branch.outputs.branch }}" >> $GITHUB_STEP_SUMMARY
          
      - name: Build
        run: mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent package org.jacoco:jacoco-maven-plugin:report

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ env.AWS_KEY }}
          aws-secret-access-key: ${{ env.AWS_SECRET }}
          aws-region: ${{ env.REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: backend
        run: |
          docker build -t backend .
          docker tag backend:latest $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_REF##*/}
          docker tag backend:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_REF##*/}
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
      # deploy:
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: assembly-fargate/src/main/resources/fargate/BACKEND_EMEA_TEST/task.json
          service: ${{ env.SERVICE_NAME }}
          cluster: ${{ env.CLUSTER_NAME }}
          wait-for-service-stability: true

      - name: Run smoke tests
        if: ${{ env.SKIP_SMOKE_TESTS == 'false' }}
        run: |
          mvn -f pom.xml clean test -P st,BACKEND_EMEA_TEST
  pipelinePassed:
    runs-on: ubuntu-latest
    needs: buildTestDeploy
    steps:
      - name: All passed
        run: |
          echo "All steps have passed."
```

2. Run the pipeline to deploy application to ECS Fargate

# Test cURLs
1. Go to AWS -> EC2 -> Load Balancers
2. Grab DNS of your Load Balancer
3. Execute below cURLs (please adjust URL)

Create test measurement

```bash
curl -vk 'http://myapp-lb-564621670.eu-central-1.elb.amazonaws.com/device/v1/test' \
--header 'Content-Type: application/json' \
-u userEMEATest:welt \
--data '{
    "type": "test",
    "value": -510.190
}'
```

Retrieve mesurements

```bash
curl -vk http://myapp-lb-564621670.eu-central-1.elb.amazonaws.com/device/v1/test -u userEMEATest:welt
```

