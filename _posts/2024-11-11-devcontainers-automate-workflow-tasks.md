---
title: "DevContainers Mastered: Automating Manual Workflows with VSCode Tasks - Part 3/3"
date: 2024-11-11
excerpt: "Turn VSCode into your own CI/CD engine - automate repetitive tasks, improve code quality, and ensure consistency for your whole team."
tags:
- vscode
- tasks
- python
- devcontainer
- devops
image: /assets/graphics/2024-11-11-devcontainers-automate-workflow-tasks/thumbnail-origami-cranes.jpg
pin: false
---

As developers, we love to automate everything. We set up CI/CD pipelines to streamline the entire lifecycle of our projects, but when it comes to our local environments, we still find ourselves repeating the same manual tasks. DevContainers help bring local environments closer to Infrastructure as Code, but it still leaves room for manual steps — like updating dependencies or triggering unit tests before we start coding.

This is where VSCode Tasks can make a real difference. By automating repetitive actions directly within the IDE, you can set up tasks to trigger tools like Poetry for managing dependencies, Pytest for running tests, or even code quality tools that lack official extensions. With VSCode Tasks, these actions run automatically, turning your local environment into a automated workflow, much like a local CI/CD pipeline.

## Setting up VSCode Tasks

Start by creating a `tasks.json` file in the `.vscode` folder of your project. This file defines tasks that can be run manually or triggered automatically based on specific events within VSCode. Here’s a simple example of a `tasks.json` setup:

```json
{
    // https://code.visualstudio.com/docs/editor/tasks#vscode
    "version": "2.0.0",
    "tasks": [
        // Poetry
        {
            "label": "Install Dependencies",
            "type": "shell",
            "command": "poetry install",
            "args": [
                "--no-interaction",
                "--with=test"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        // Pytest
        {
            "label": "Run Tests",
            "type": "shell",
            "command": "pytest",
            "args": [
            "./",
            "-c=../.tools/pytest.ini"
            ],
            "group": {
                "kind": "test"
            },
            "dependsOn": [
                "Install Dependencies"
            ],
            "runOptions": {
                "runOn": "folderOpen"
            },
            "presentation": {
                "echo": true,
                "reveal": "never",
                "revealProblems": "onProblem",
                "focus": false,
                "panel": "dedicated",
                "showReuseMessage": true,
                "clear": false
            }
        }
    ]
}
```
{: file='devcontainer.json' }

This configuration defines two tasks: one for installing dependencies and another for running tests, both triggered via shell comma

![actions-view](/assets/graphics/2024-11-11-devcontainers-automate-workflow-tasks/vscode_tasks_ui_screenshot.webp)

## Organising and Grouping Tasks

Tasks in VSCode can be grouped to organize them by category.

- **Install Dependencies** is grouped under the `build` category.
- **Run Tests** is grouped under the `test` category.

The `isDefault` setting ensures that the default task for a group runs when triggered. You can create custom groups as well, depending on your workflow needs.


## Customising Task Output with Presentation Options

The `presentation` section in the `tasks.json` file allows you to control how task outputs appear in VSCode. Here are the key attributes to configure the presentation:

    **echo**: If `true`, the task's command output is echoed in the terminal.
    **reveal**: Controls when and how the terminal window is shown, `never` prevents the terminal from opening automatically.
    **revealProblems**: Determines when to reveal problems in the terminal output (e.g., on encountering a problem).
    **focus**: If set to `true`, focuses the terminal window on task execution.
    **panel**: Defines where the terminal output will appear. Options include `shared` or `dedicated`, where `dedicated` ensures a separate terminal panel for the task.
    **clear**: If `true`, the terminal is cleared before running the task.

In the **Run Tests** task example, the presentation ensures that the output is shown in a dedicated terminal panel without disrupting your workflow, and any problems will be revealed as needed.

## Automatically start Tasks with Event Triggers

VSCode allows tasks to be triggered automatically based on events like `folderOpen`, `fileSave`, or `fileChanges`. For example, running dependency management or tests on `folderOpen` ensures everything is up-to-date before starting development. These tasks run in the background without interrupting your workflow, only notifying you if there are issues. It guarantees you everything is working without causing delay.

## Using VSCode Tasks in DevContainers

The documentation of DevContainers does not mention support for VSCode Tasks. However, adding a section of tasks to the devcontainer.json does work without issue. [Add tasks support to devcontainer.json #9951
](https://github.com/microsoft/vscode-remote-release/issues/9951)

DevContainer.json example with VSCode tasks:

```json
// https://containers.dev/implementors/json_reference/
{
    "name": "hello_world",
    "image": "python:3.12",
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python", // https://marketplace.visualstudio.com/items?itemName=ms-python.python
            ],
            "settings": {
                "tasks": {
                    "version": "2.0.0",
                    "tasks": [
                        // Poetry
                        {
                            "label": "Install Dependencies",
                            "type": "shell",
                            "command": "poetry install",
                            ...
                        },
                        // Pytest
                        {
                            "label": "Run Tests",
                            "type": "shell",
                            "command": "pytest",
                            ...
                        }
                    ]
                }
            }
        }
    }
}
```
{: file='devcontainer.json' }

## Using Code Quality Tools as Custom Tasks

While some code quality tools don’t have official extensions in VSCode, you can still integrate them by creating custom tasks in the `tasks.json` file. These tools can be triggered manually, without being part of an automated workflow. By defining the tool as a task, you can run it anytime directly from the IDE, using the top menu (Terminal > Run Task) or the command palette.

This approach gives you the flexibility to use any external tool (like linters or static analysis tools) that may not have a native VSCode extension, without making them part of your automated process. You retain full control over when to trigger these tasks, making them as flexible as you need.

## Explore More

Dive deeper into code quality and CI/CD best practices by exploring the rest of my mini-series on code quality tools. For additional configurations, tools, and best practices to elevate your workflow, check out my DevOps Toolkit repo on GitHub.
