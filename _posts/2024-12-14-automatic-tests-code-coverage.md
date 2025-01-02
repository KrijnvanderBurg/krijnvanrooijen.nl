---
title: "Automatically Generate and Visualize Python Code Coverage in VSCode"
date: 2024-12-16
excerpt: "How to automate code coverage and visualize results directly in VSCode for Python, improve your tests with real-time coverage insights."
tags:
- code coverage
- pytest
- tests
- vscode tasks
- devcontainer
image: /assets/graphics/2024-12-14-automatic-tests-code-coverage/thumbnail-code-coverage-veiled.png
pin: false
---

Many teams already have code coverage set up in their CI/CD pipelines, often through tools like pytest or SonarQube. However, accessing these results usually involves waiting for pipeline runs to complete or navigating to external dashboards, creating a barrier for developers. This barrier discourages frequent checks on coverage, with many only reviewing it when a pipeline fails due to insufficient coverage. This article shows how to automate test coverage with the pytest framework, integrate it with VSCode, and leverage the `Coverage Gutters` extension for real-time feedback. 


## Pytest and the Coverage Plugin

pytest is a popular testing framework in Python that can be extended with the coverage plugin. This plugin generates code coverage reports, highlighting which lines of code are tested and which are not. The plugin supports multiple formats, making it versatile for integration with various tools and dashboards, the most common format is junit XML.

To use pytest with the coverage plugin, run the following command:

```bash
pytest ./ ./tests/ --cov=./ --cov-report=xml:coverage.xml -cov-config=.coveragerc
```
- `-cov=./`: The path to the code, which the tests are compared against for coverage.
- `-cov-report=xml:coverage.xml`: Generates a coverage report in XML format and saves it to `coverage.xml`.
- `-cov-config=.coveragerc`: Specifies the path to the `.coveragerc` configuration file.

For more details on adding code quality tools like pytest, check out my earlier post: [*DevContainers Improved: Integrating Code Quality Checks for Continuous Feedback — Part 2/3*](https://krijnvanderburg.medium.com/add-code-quality-tools-in-your-ide-840df78c64d5).

## Automating Code Coverage in Your Development Workflow

Automating code coverage removes the barriers that come with relying on external CI/CD tools or dashboards. Instead of waiting for pipeline runs to complete or navigating to separate reports, developers can access immediate, consistent feedback within their development environment. This eliminates the delay that often discourages developers from checking code coverage frequently. By integrating coverage directly into the IDE, developers can instantly see which lines are covered and which aren’t, improving test quality and speeding up the debugging process.

There are two main ways to automate code coverage. Via the VSCode python extension that adds running Pytest thourhg the UI sidebar. And by leveraging VSCode Tasks to run a pytest coverage command on certain event triggers. 


### Coverage via the VSCode python/pytest extension

When the Python/Pytest extension is triggered through the VSCode UI to run tests, it executes a pytest command in the background. This command can be further customized by adding additional options via the pytestArgs setting. Specifically, the .coveragerc configuration file is included, and the desired output format for the coverage report is defined, as previously discussed in the pytest section.

```json
"customizations": {
    "vscode": {
        "extensions": [],
        "settings": {
            "python.testing.pytestEnabled": true,
            "python.testing.pytestArgs": [
                "./src/",
                "./tests/",
                "--cov=./",
                "--cov-report=xml:./coverage.xml",
                "--cov-config=${workspaceFolder}/../.tools/.coveragerc"
            ]
        }
    }
}
```
{: file='devcontainer.json' }

This configuration ensures coverage generation is consistent across both local and CI/CD environments.

### Coverage via VSCode Tasks

VSCode tasks is a simple but effective way of automating any type of tool or command and can be configured in both `.vscode` and `.devcontainer` configurations. The pytest coverage command can be set to trigger automatically on various VSCode events, e.g. when the project is opened. This automation removes the need to manually start tests and keeping coverage up to date. This approach makes it a zero-effort solution for teams to consistently keep track of coverage without thinking about it.

For more on how to use VSCode tasks, and how they work in a DevContainer setup, check out my earlier article: [*DevContainers Mastered: Automating Manual Workflows with VSCode Tasks — Part 3/3*](https://medium.com/@krijnvanderburg/how-i-automate-my-entire-ide-vscode-akin-to-cicd-992568ee7fb5). The configuration below runs pytest coverage whenever the project folder is opened:

```json
"tasks": {
    // https://code.visualstudio.com/docs/editor/tasks#vscode
    "version": "2.0.0",
    {
        "label": "pytest coverage",
        "type": "shell",
        "command": "pytest",
        "args": [
            "./src/",
            "./tests/",
            "--cov=./",
            "--cov-report=xml:./coverage.xml",
            "--cov-config=../.tools/.coveragerc"
        ],
        "dependsOn": [
            "Poetry Install"
        ],
        "runOptions": {
            "runOn": "folderOpen"
        }
    }
}
```

### Coverage Gutters

Generating a coverage file is only part of the solution. No engineer is going to read XML to determine which lines have and have not been covered. It is worse than waiting for CI/CD and getting a file with just the overall percentage of coverage. The Coverage Gutters extension for VSCode solves this by displaying coverage data directly in the editor.

This extension uses the coverage file to provide a real-time, color-coded view of which lines are covered and which aren’t, right in the gutter next to the code. As the file updates, the coverage visualization refreshes automatically, making it easy to spot untested areas.

Key features of Coverage Gutters include:

- **Watch Mode**: Tracks changes to coverage files and updates the editor in real time.
- **Report Previews**: Displays HTML summaries of coverage reports within the IDE.

However, the extension doesn’t start watching for coverage files automatically by itself. After opening a project, use the Command Palette or the “Watch” button in the bottom bar of VSCode to activate watch mode. Or trigger the `watch` command via a VSCode task, which has a `dependsOn` the pytest coverage task. Once enabled, the coverage display updates whenever the coverage.xml file changes. 
```json
"tasks": {
    // https://code.visualstudio.com/docs/editor/tasks#vscode
    "version": "2.0.0",
    {
        "label": "pytest coverage",
        "type": "shell",
        "command": "pytest",
        "args": [
            "./src/",
            "./tests/",
            "--cov=./",
            "--cov-report=xml:./coverage.xml",
            "--cov-config=../.tools/.coveragerc"
        ],
        "dependsOn": [
            "Poetry Install"
        ],
        "runOptions": {
            "runOn": "folderOpen"
        }
    },
    {
        "label": "coverage-gutters watch",
        "dependsOn": [
            "pytest and coverage"
        ],
        "presentation": {
            "reveal": "never"
        },
        "command": [
            "${command:coverage-gutters.watchCoverageAndVisibleEditors}"
        ],
        "problemMatcher": []
    },
}
```

By integrating coverage data directly into the IDE, Coverage Gutters eliminates the need for external tools and encourages better testing practices. Developers can focus on addressing gaps in coverage without disruptions, making it an essential addition to any efficient workflow.


## Coverage as Continuous Improvement

Both methods of generating a coverage can be combined into an effortless setup. The Python extension handles quick, manual test runs via the UI, while VSCode tasks ensure automated coverage generation during events like opening a project. This makes sure that coverage data is always fresh with minimal effort.

With coverage.xml in place, the Coverage Gutters extension displays color-coded highlights directly in the editor, showing exactly which lines are covered and which aren’t. Updates happen in real time as the file changes—no need to parse XML or rely on external tools. This setup eliminates friction, keeps coverage visible, and ensures consistent testing practices throughout development.
