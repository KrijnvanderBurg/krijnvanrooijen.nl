---
title: "Distribute Tests with Pytest-Split for Faster CI/CD Execution — Zero-BS #1"
date: 2024-09-23
excerpt: "Speed up test suite execution by using pytest-split to distribute tests across multiple CI/CD agents. Pytest-split is a plugin for the pytest framework that allows users to split their test suite into smaller groups for parallel execution, reducing overall test runtime."
tags:
- pytest
- unit test
- CICD
- Azure Pipelines
- Distributed
image: /assets/graphics/2024-09-23-distribute-tests-with-pytest/azure_devops_pipeline_screenshot.webp
pin: false
---

Speed up test suite execution by using pytest-split to distribute tests across multiple CI/CD agents. This example uses Azure Pipelines, but the method applies to any CI/CD platform that supports parallel execution.

## Pytest-split
`Pytest-split` is a plugin for the `pytest` framework that allows users to split their test suite into smaller groups for parallel execution, reducing overall test runtime. It can split tests based on the number of groups or specific markers, making it ideal for large projects. Installation is straightforward via `pip install pytest-split`, and it integrates well with other plugins like `pytest-xdist` for parallelization.

## Basic implementation
The Azure Pipeline snippet below is a basic implementation to distribute pytest-split to multiple Azure DevOps Agents.

The job `Test` is set to run in parallel across five agents. By combining the pytest-split arguments `--splits` and `--group` with the Azure Pipeline variables `$(System.JobPositionInPhase)` and `$(System.TotalJobsInPhase)`, the tests can be split dynamically based on the amount of parallel agents.

- `--splits` specifies the total number of test groups to create, allowing pytest-split to evenly divide the test suite. By using `$(System.TotalJobsInPhase)`, it dynamically assigns the total number of parallel jobs, ensuring that each worker gets an equal share of the tests to execute.
- `--group` determines which specific group of tests a worker will run. It uses `$(System.JobPositionInPhase)` to identify the worker's position in the job queue, ensuring that each worker executes only its assigned tests from the split.

```yaml
jobs:
  - job: Test
    displayName: Pytest-split group 
    strategy:
      parallel: 5
    steps:
      - task: UsepythonVersion@0
        displayName: Agent set Python
        inputs:
          versionSpec: 3.12
          architecture: x64

      # ... Install tests dependencies, aka requirements.txt or pyproject.toml

      - script: |
          pip install pytest pytest-cov pytest-azurepipelines pytest-xdist pytest-split

          pytest ./<path to tests> \
          --cov=<coverage files path> \
          --cov-report=xml \
          --junit-xml=$(Build.StagingDirectory)/JUNIT-TEST.xml \
          --splits=$(System.TotalJobsInPhase) \
          --group=$(System.JobPositionInPhase)
        displayName: Run pytest
```

## The problems with the basic implementation:

Distributing tests introduces several challenges:

1. **Multiple Output Files**: Each job generates its own pytest output file and coverage report, which are published separately in Azure DevOps, complicating results interpretation.
2. **Imbalanced Load**: If long-running tests are assigned to the same agent, overall waiting time may not decrease, undermining the benefits of distribution. Ensure these tests are evenly distributed across agents for optimal performance.

## Final and complete implementation:

To address the above challenges, all pytest and coverage output files are uploaded as separate artifacts from each job. These artifacts are then merged into a single file using `coverage combine`, which is subsequently published to Azure DevOps.

```yaml
jobs:
  - job: Test
    displayName: Pytest-split group 
    strategy:
      parallel: 5
    steps:
      - task: UsepythonVersion@0
        displayName: Agent set Python
        inputs:
          versionSpec: 3.12
          architecture: x64

      # ... Install tests dependencies, aka requirements.txt or pyproject.toml

      - script: |
          pip install pytest pytest-cov pytest-azurepipelines pytest-xdist pytest-split

          pytest ./<path to tests> \
          --cov=<coverage files path> \
          --cov-report=xml \
          --junit-xml=$(Build.StagingDirectory)/JUNIT-TEST.xml \
          --splits=$(System.TotalJobsInPhase) \
          --group=$(System.JobPositionInPhase)
        displayName: Run pytest
        env:
          COVERAGE_FILE: $(Build.StagingDirectory)/coverage/$(System.JobPositionInPhase).coverage

      - publish: $(Build.StagingDirectory)/coverage/$(System.JobPositionInPhase).coverage
        displayName: Publish $(System.JobPositionInPhase).coverage
        artifact: $(System.JobPositionInPhase).coverage
  
  - job: MergeCoverage
    displayName: Merge coverage
    dependsOn: TestSplit
    condition: succeeded()
    steps:
      - task: DownloadPipelineArtifact@2
        inputs:
          path: $(System.DefaultWorkingDirectory)

      - script: |
          pip install coverage

          coverage combine $(System.DefaultWorkingDirectory)/**/*.coverage

          coverage xml -o $(Build.StagingDirectory)/coverage.xml
        displayName: Merge coverage

      - task: PublishCodeCoverageResults@2
        displayName: Code overage publish results
        inputs:
          codeCoverageTool: Cobertura
          summaryFileLocation: $(Build.StagingDirectory)/coverage.xml
          searchFolder: $(Common.TestResultsDirectory)
          pathToSources: ${{ parameters.coveragePath }}
          failIfCoverageEmpty: true
```

<br>

![actions-view](/assets/graphics/2024-09-23-distribute-tests-with-pytest/azure_devops_pipeline_screenshot.webp)