---
title: "How to enforce Code Quality standards using Azure Pipelines"
date: 2024-12-11
excerpt: "Learn how to enforce code quality standards with CI/CD pipelines for tools like Ruff, ensuring consistency and security in your software development."
tags:
- cicd
- azure pipelines
- code quality
- ruff
- python
- devops
image: /assets/graphics/2024-12-11-enforce-code-quality-via-cicd/thumbnail-scale-set-python-security.png
pin: false
---

Setting Code Quality standards is non-negotiable software development. Even more so for corporate environments where software products slowly mould into legacy, living an unintentionally and unhealthy long lifespan.

In a [previous article](https://medium.com/@krijnvanderburg/add-code-quality-tools-in-your-ide-840df78c64d5), I covered adding code quality tools to IDEs in DevContainers for consistent standards across teams with minimal setup. However, IDE-based tools are not foolproof; standards need enforcement beyond the developer's environment; code quality checks integrated into the CI/CD pipeline establish quality gates that halt deployment until standards are met. This article uses Azure Pipelines as an example, but the approach applies to any CI/CD platform.

## Integrating Tools into CI/CD Pipeline

Most code quality tools lack native Azure Pipeline tasks, but can still be integrated through command-line commands for installing, configuring, and running checks. Ruff will serve as an example for integrating a CI/CD quality gate.
The foremost challenge is to maintain uniformity between the local development and CI/CD environments. Most code quality tools support configuration files, ensuring a consistent approach across environments. Centralizing configurations in Git allows both developers and CI/CD pipelines to use the same setup. Differences between configurations can result in failed CI/CD runs, or worse, passing while it should fail.

## Example pipeline with Ruff

The following example demonstrates how to set up Ruff in a CI/CD pipeline with Azure Pipelines. Three elements are critical: mirroring the local configuration, specifying a target folder for analysis, and ensuring the tool exits with errors if quality checks fail. Ruff can be installated via  pip install, invoking ruff check for analysis, and using flags to customize behavior:
- `-config` points to the configuration file.
- `-diff` and `-show-fixes` display issues and their solutions, making failures self-explanatory rather than cryptic.
- `-no-fix` disables auto-fixing, only to analyze without modifying code. This ensures CI/CD serves only as a gate, not as a formatter.
- `-verbose` provides detailed feedback, although verbosity varies by tool. Use it if the output provides meaningful context.

The Azure Pipeline template below installs Ruff and applies it to the designated folders or files, reporting any issues it finds.

*Printing the tool's version in the CI/CD log enables fast debugging, ensuring consistency with the expected environment version. Printing configurations is also helpful for verification.*

```yaml
parameters:
- name: targetPath
  type: string

- name: configFilepath
  type: string
  default: $(Build.Repository.LocalPath)/path/to/ruff.toml

steps:
- script: |
    pip install ruff --quiet

    ruff --version

    ruff check ${{ parameters.targetPath }} \
    --config ${{ parameters.configFilepath }} \
    --no-fix \
    --diff \
    --show-files \
    --verbose
  displayName: Ruff check
```
{: file='devcontainer.json' }

This same process can be applied to any command line tool, not just Ruff. By following similar steps, tools like MyPy, Flake8, and Pylint can be added, creating a comprehensive CI/CD pipeline that that enforces code standards effectively across multiple tools.

## Extending the Example to Other Tools

The example with Ruff highlights the general steps for integrating a code quality tool into a CI/CD pipeline. Combining tools like Flake8, MyPy, Ruff, and Bandit creates a robust code quality pipeline.

- **MyPy**: For type checking in Python projects, MyPy ensures type correctness, preventing runtime errors due to mismatched data types.
- **Flake8**: A linter that enforces PEP 8 compliance, checks for code style issues, and identifies potential errors.
- **Bandit**: Focused on security, Bandit scans Python code for common vulnerabilities, to proactively address code any security risks.

![actions-view](/assets/graphics/2024-12-11-enforce-code-quality-via-cicd/cicd-pipeline-screenshot.png)

## Optimizing the surrounding CI/CD Process

When enforcing standards in CI/CD, design the pipeline to provide full clarity for users without causing unnecessary halts. If multiple quality gates are integrated run all tools regardless of whether any one of them fails. This ensures users receive the complete list of issues at once, avoiding the frustration of fixing one tool's output, restarting the pipeline, and then fixing the next tool's output.

Set the property `continueOnError: true` for each quality gate job in the CI/CD pipeline. With this configuration, the pipeline will continue executing each quality gate job even if one or more fail. However, the pipeline's final build or deployment stage will only proceed if all gates pass successfully. This setup captures all issues in a single pipeline run, saving time and providing the feedback of all tools without repeated restarts.

# Explore More
For more on code quality standards and CI/CD best practices, explore other articles in this series. For a full toolkit of code quality tools and configurations, visit my [GitHub: DevOps Toolkit](https://github.com/KrijnvanderBurg/DevOps-Toolkit).
