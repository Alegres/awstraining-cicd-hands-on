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
