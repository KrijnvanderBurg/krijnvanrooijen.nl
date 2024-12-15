---
title: "DevContainers Improved: Integrating Code Quality Checks for Continuous Feedback - Part 2/3"
date: 2024-10-22
excerpt: "Improve your workflow with DevContainers! Integrate code quality checks in VSCode for real-time feedback and error-free code."
tags:
- code quality
- devcontainer
- containers
- vscode
- ruff
image: /assets/graphics/2024-10-22-devcontainers-add-code-quality-tools/thumbnail-ship-approaching-lighthouse.png
pin: false
---

Code quality tools are essential for maintaining consistency and catching errors in a codebase. While Pre-commit hooks are a common solution, they only catch issues after the work is finished, leading to frustration and inefficiency. By integrating code quality tools directly into the IDE, errors can be identified and corrected instantly, streamlining the workflow.

Pre-commit hooks are commonly used to catch issues before code is pushed to the repository. However, they only run during commits, resulting in a list of problems to fix after the work is completed. A more friendly and efficient approach is to integrate code quality tools directly into the IDE for instant feedback while programming.

This example uses VSCode with DevContainers to automatically integrate tools like Ruff, Black, Mypy, and Flake8 into your workflow. Similar functionality is available in other IDEs.

## VSCode extensions

All of the aforementioned tools have VSCode extensions that automate execution and integrate output directly into the VSCode interface. These extensions can be added to the DevContainer using their VSCode Marketplace IDs. When the DevContainer is built, the extensions are installed automatically, creating an environment akin to infrastructure as code. Developers can pull the repository and run the container without manual configuration.

Suppose the team wants to implement an autoformatter for Python code. Two well-known choices are Black and Ruff, the latter being a relatively new tool with outstanding performance. The documentation for Ruff specifies how Ruff can be configured using a ruff.toml file; consider the basic example below.

## Ruff formatter example

The `ruff.toml` file defines how the tool behaves, including what file types to exclude (`exclude`), maximum line length (`line-length`), and indentation style (`indent-width`). The `[lint]` section specifies which linting rules to enforce, and the `[format]` section sets preferences like quote style and line endings.

```toml
exclude = [".git", ".venv", "node_modules"]

line-length = 120
indent-width = 4
target-version = "py311"

[lint]
select = ["E4", "E7", "E9", "F"]  # Enable specific codes
fixable = ["ALL"]                 # Allow fixing all rules

[format]
quote-style = "double"            # Use double quotes
indent-style = "space"            # Use spaces for indentation
line-ending = "auto"              # Auto detect line endings
```

The Marketplace page for the VSCode extension for Ruff details the available configurations, including the path to the configuration file. The `ruff.toml` file can be specified via the lint and format args settings in the `devcontainer.json` file:

```json
// https://containers.dev/implementors/json_reference/
{
    "name": "hello_world",
    // ...
    "customizations": {
        "vscode": {
            "extensions": [
                "charliermarsh.ruff", //https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff
            ],
            "settings": {
                //
                // Python
                //
                "editor.defaultFormatter": "charliermarsh.ruff",
                "editor.formatOnSave": true,
                "editor.codeActionsOnSave": {
                "source.fixAll": "explicit",
                "source.organizeImports": "explicit"
                //
                // Ruff - https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff
                //
                "ruff.lint.args": ["--config=${workspaceFolder}/path/to/ruff.toml"],
                "ruff.format.args": ["--config=${workspaceFolder}/path/to json/ruff.toml"],
                "ruff.organizeImports": true,
                "ruff.fixAll": true
            }
        }
    }
}
```
{: file='devcontainer.json' }

Every team member using this DevContainer will automatically have a linter and formatter included in their environment, detailing all identified issues in the integrated UI of VSCode under the Problems tab.

![actions-view](/assets/graphics/2024-10-22-devcontainers-add-code-quality-tools/vscode_code_quality_tools_screenshot.png)

Store the configuration file and `devcontainer.json` in Git. When team members pull the repository and rebuild the container, the Ruff autoformatter will work automatically, requiring no manual setup. This ensures the entire team formats code according to the specified config file, achieving standardization effortlessly. These configuration files can also serve as starting points for discussions on additional tools to incorporate and settings to enable, given that the workflow remains consistent for everyone.

The result is a development environment where code quality tools integrate directly into the IDE, providing instant feedback and keeping the team aligned on standards. With tools like Ruff, Black, Mypy, and Flake8 built into the DevContainer, productivity increases and the codebase remains solid without the hassle.

## Explore more

The process of adding tools becomes iterative; additional extensions can be included in the same manner as Ruff. I have already added well-known code quality tools such as Black, Mypy, and Flake8 and many more to the DevContainer. For a comprehensive list of these tools with configurations, checkout my GitHub: [github.com/KrijnvanderBurg/DevOps-Toolkit](https://github.com/KrijnvanderBurg/DevOps-Toolkit).

While not all tools may have an extension, unsupported tools can still be added using VSCode Tasks, more on this in a future article.
