---
title: "Why I Stopped Using Dependency Managers (They Hide User Install Risks)"
date: 2024-12-24
excerpt: "Understanding why dependency tools like Poetry and Pip hide real-world user issues, and how building and installing your own package can reveal existing problems."
tags:
- dependency
- dependency manager
- pip
- poetry
- wheel
- artifact
- build
image: /assets/graphics/2024-12-15-why-i-stopped-using-dependency-managers/thumbnail-book-as-bookcase.png
pin: false
---
Every application with dependencies exists in two worlds: the strictly controlled stability of a developer’s environment, and the unpredictable dependency landscape of a user’s installation. This disconnect is especially pronounced in ecosystems like Python, where no single standardized toolset exists, and developers must navigate an ever-evolving landscape of varying standards and tools.

To ensure stability and reproducibility, developers traditionally lock dependency versions during development. For example, in Python, this might involve using a `poetry.lock` file with pinned versions pushed to git. While this guarantees consistency within the development team, it creates a critical blind spot: developers are shielded from the real-world challenges users face when installing the application. Users encounter dynamically resolved dependencies, influenced by version ranges and what’s available at the time of installation—including newer releases or versions that developers may never have tested.

This article challenges the current standard approach to installing dependencies. While locking dependencies ensures stability in development, it doesn’t reflect the real-world experience of users. Is this the right approach, or is it time for a shift in how we install dependencies?

### The Established approach in Python
Python’s package management typically relies on tools like pip and Poetry, which aim to manage dependencies and ensure reproducibility. However, in practice, only one resolved set of dependency versions is tested during development—regardless of whether version ranges are specified.

#### pip and requirements.txt
With pip, dependencies are defined in a requirements.txt file, using pinned versions or version ranges:

```python
numpy==1.21.0
pandas>=1.3.0,<2.0.0
```
{: file='requirements.txt' }

Developers typically install dependencies once during setup with `pip install -r requirements.txt`. While version ranges offer flexibility, the installation resolves specific versions, which are used throughout development unless a problem forces reinstallation. In practice, this means developers often work with a single set of resolved dependencies, leaving broader version ranges untested.

#### Poetry and pyproject.toml / poetry.lock
Poetry improves reproducibility by using a `pyproject.toml` and generating a `poetry.lock` file, locking all dependencies to exact versions. Example of a `pyproject.toml`:

```toml
[tool.poetry.dependencies]
numpy = ">=1.21.0,<2.0.0"
pandas = ">=1.3.0,<2.0.0"
```
{: file='pyproject.toml' }

The `poetry.lock` is pushed to git and shared across the team. This ensures all developers work with the same versions and avoid inconsistencies. Meaning, only the locked versions are tested unless dependencies are explicitly updated or the lock file is recreated.

## The Real-World Challenge
Despite the flexibility of version ranges, both pip and Poetry workflows often result in developers testing only one resolved set of dependency versions during development. This leaves little understanding of how different combinations of dependencies might behave in real-world scenarios when users install the application.

This gap between development and user experiences can lead to subtle bugs or regressions. For example, a package may specify `pandas>=1.3,<2.0`, but during development, the `poetry.lock` file might pin `pandas` to version `1.4.3`, which works perfectly in tests. However, when a user installs the package months later, they may get `pandas 1.5.3`, which could introduce a bug that wasn’t present during development.

### Why Version Pinning is Problematic
Pinning dependency versions is crucial for reproducibility, especially in team settings. It ensures everyone works with the same versions, avoiding surprises from unexpected updates. However, this approach can also conceal issues that users might face when ther installation resolved to different dependencies.

To address this, one possible solution is to test against multiple dependency versions. Tools like Tox and Nox allow developers to define test matrices for different Python versions and dependency versions. However, testing all combinations quickly becomes infeasible. The Cartesian product, an explosion of possibilities, makes testing impractical or even impossible considering testing performance. Therefore, exhaustive testing is not a feasible solution.

## A Practical Shift: Install your own package
To address these issues, I have adapted my workflow to better reflect the user experience. Instead of simply installing dependencies, I package my application (in this case, for Python) into a `.whl` file locally. I then install this wheel using pip, which installs the dependencies just like a user would. This approach mirrors the way users will interact with the application, allowing me to identify issues that might otherwise go unnoticed in a purely development environment. If I don't experience any dependency issues, but my colleague who does, then that is a success.

But this isn’t a practical workflow for everyday development. You can’t expect developers to constantly package and install their application manually every time they want to run tests. This is where automation comes into play. In a previous article, I discussed how to automate your local development environment to reduce manual commands. I have built upon this automation and have mimiced my user experience, effortlessly.

### Automating with VSCode Tasks
Below is a sample VSCode task configuration that automates the process of dependency installation and testing, while also incorporating the packaging step. These VSCode tasks can be integrated into a devcontainer to standardize the workflow for your entire team.

For more on automating local development environments, check out my previous articles such as: *[DevContainers Mastered: Automating Manual Workflows with VSCode Tasks](https://medium.com/@krijnvanderburg/how-i-automate-my-entire-ide-vscode-akin-to-cicd-992568ee7fb5)*.

```json
"tasks": {
    // https://code.visualstudio.com/docs/editor/tasks#vscode
    "version": "2.0.0",
    "options": {},
    "tasks": [
        {
            "label": "build package",
            "type": "shell",
            "command": "poetry",
            "args": [
                "build",
                "--output",
                "./dist/"
            ],
            "group": "build",
            "presentation": {
                "showReuseMessage": false
            },
            "problemMatcher": []
        },
        {
            "label": "install package",
            "type": "shell",
            "command": "pip",
            "args": [
                "install",
                "./dist/*.whl"
            ],
            "dependsOn": [
                "build package"
            ],
            "presentation": {
                "panel": "shared"
            },
            "group": "build",
            "problemMatcher": []
        }
    ]
}
```

## My thoughts in summary
Traditional workflows lock dependency versions to ensure stability during development, but this approach does not reflect the user’s experience when they install the package. By packaging and installing my own application locally, I can identify issues that might otherwise go unnoticed. With automation tools like VSCode tasks, this process becomes effortless and helps simulate the actual user experience.
