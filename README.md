# Hello world workflow
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

# Custom composite action
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

Please adjust path to the action.

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
                 "AWS_ACCOUNT": "467331071075",
                 "PROFILE": "backend-test",
                 "REGION": "eu-central-1",
                 "CLUSTER_NAME": "backend-ecs-test",
                 "SERVICE_NAME": "backend",
                 "TASK_NAME": "backend"
              },
              "BACKEND_US_TEST": {
                 "HUB": "US",
                 "STAGE": "TEST",
                 "AWS_ACCOUNT": "467331071075",
                 "PROFILE": "backend-test",
                 "REGION": "us-east-1",
                 "CLUSTER_NAME": "backend-ecs-test",
                 "SERVICE_NAME": "backend",
                 "TASK_NAME": "backend"
              }
            }
```

# Terraform pipeline
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

# Test cURLs
First, grab DNS of Load Balancer from AWS.

Create test measurement

```bash
curl -vk 'http://myapp-lb-564621670.eu-central-1.elb.amazonaws.com/device/v1/test' \
--header 'Content-Type: application/json' \
-u testUser:welt \
--data '{
    "type": "test",
    "value": -510.190
}'
```

Retrieve mesurements

```bash
curl -vk http://myapp-lb-564621670.eu-central-1.elb.amazonaws.com/device/v1/test -u testUser:welt
```

