---
title: "Local Spark Cluster DevContainer"
date: 2025-04-20
excerpt: "Out-of-the-box working local Spark cluster with PySpark and Jupyter notebooks as DevContainer for streamlined data engineering workflows."
tags:
- DevContainer
- Data Engineering
- PySpark
- Databricks
- Spark
image: /assets/graphics/2025-04-20-local-spark-cluster-devcontainer/thumbnail.jpg
pin: false
---

Developing and testing Spark applications typically requires a complex setup process with multiple dependencies, network configurations, and environmental considerations. Setting up a local Spark cluster often involves manual installation steps, configuration tweaking, and troubleshooting connection issues. This complexity can significantly slow down development cycles and create barriers to entry for newcomers to Spark development.

By combining DevContainers with Docker Compose, data engineers can now deploy a fully functional local Spark cluster with minimal effort. This article introduces a ready-to-use DevContainer setup that eliminates the hassle of configuring and maintaining a local Spark environment.

## Architecture Overview

The local Spark cluster DevContainer consists of multiple interconnected services:

- **DevContainer**: The primary development environment with all necessary tools and libraries
- **Spark Master**: The central coordination node for the Spark cluster
- **Spark Workers**: Multiple worker nodes that perform the actual computation tasks
- **Spark History Server**: A web interface for monitoring and debugging completed Spark jobs

Each component is containerized and configured to work together seamlessly, creating a production-like environment directly on the developer's machine.

## Getting Started

The setup process has been simplified to require minimal manual intervention. After cloning the repository, simply open it in VSCode with the DevContainers extension installed and select "Reopen in Container." The entire cluster will be automatically provisioned and configured.

```bash
git clone https://github.com/KrijnvanderBurg/local-spark-cluster
cd local-spark-cluster
code .
```

Once VSCode prompts to reopen in container, the entire environment will be built and started automatically. This includes:

1. Building all necessary Docker images
2. Starting the Spark master and worker nodes
3. Configuring network communication between containers
4. Setting up Python environments with PySpark

## Docker Compose Configuration

The heart of this setup is the Docker Compose file, which orchestrates all the services required for a functional Spark cluster:

```yaml
version: '3.4'

# Worker template that will be reused
x-spark-worker-template: &spark-worker-template
  image: apache/spark:${SPARK_VERSION}
  command: "/opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:${SPARK_MASTER_PORT}"
  environment:
    - SPARK_PUBLIC_DNS=localhost  # For UI URLs to be accessible from host
    - SPARK_LOCAL_IP=spark-worker-1  # Will be overridden by specific workers
  volumes:
    - spark-logs:/opt/spark/logs
    - spark-events:/opt/spark/events
    - ./spark-defaults.conf:/opt/spark/conf/spark-defaults.conf
    - ../../:/workspace
  networks:
    - spark-network
  depends_on:
    - spark-master
  restart: unless-stopped

services:
  devcontainer:
    # Development environment container
    # ...existing code...
  
  spark-master:
    # Spark master node
    # ...existing code...
  
  # Worker nodes and history server
  # ...existing code...
```

This configuration uses a worker template to avoid repetition and ensure consistency between worker nodes. The environment variables controlling Spark versions and resource allocation are defined in a separate `.env` file, making it easy to adjust the cluster configuration without changing the core setup.

## Automating PySpark Initialization

To streamline the development experience, the DevContainer includes an automatic initialization script for Jupyter notebooks:

```python
# Initialize Spark
from pyspark.sql import SparkSession

# Build the SparkSession with sensible defaults for notebook usage
spark = SparkSession.builder \
    .appName("Jupyter Notebook") \
    .master("spark://spark-master:7077") \
    .config("spark.driver.host", "devcontainer") \
    .config("spark.driver.bindAddress", "0.0.0.0") \
    .config("spark.ui.port", "4040") \
    # ...existing code...
    .getOrCreate()

# Create a SparkContext variable for backward compatibility
sc = spark.sparkContext
```

This script runs automatically when a notebook kernel starts, ensuring that a properly configured Spark session is always available without manual setup steps.

## Simplified Job Submission

Submitting Spark jobs to the cluster is made straightforward with a custom `spark-submit.sh` script:

```bash
#!/bin/bash
# Use first argument as the file to submit, or default to sample_job.py if no argument provided
FILE_TO_SUBMIT=${1}

/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --conf spark.driver.host=devcontainer \
  --conf spark.driver.bindAddress=0.0.0.0 \
  "$FILE_TO_SUBMIT"
```

The script is integrated into VSCode tasks, allowing developers to submit the current file to the cluster with a simple keyboard shortcut (Ctrl+Shift+B).

## VSCode Integration

The DevContainer is configured with VSCode settings and extensions optimized for Spark development:

```json
{
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-toolsai.jupyter",
                "ms-toolsai.datawrangler",
                "ms-python.python",
                "ms-python.vscode-pylance"
            ],
            "settings": {
                // Jupyter settings
                "jupyter.interactiveWindow.textEditor.executeSelection": true,
                "jupyter.notebookFileRoot": "${workspaceFolder}",
                // ...existing code...
                
                // VSCode Tasks
                "tasks": {
                    "version": "2.0.0",
                    "tasks": [
                        {
                            "label": "spark-submit current file",
                            "type": "shell",
                            "command": "${workspaceFolder}/.devcontainer/spark/spark-submit.sh ${file}",
                            "group": {
                                "kind": "build",
                                "isDefault": true
                            }
                        }
                    ]
                }
            }
        }
    }
}
```

This configuration enhances productivity by providing:

1. Jupyter notebook integration for interactive development
2. Data wrangling tools for easier data exploration
3. Python language support with intelligent code completion
4. Task automation for common workflows

## Benefits for Data Engineers

This DevContainer-based Spark cluster offers several advantages for data engineering teams:

1. **Consistency**: Every team member works with the exact same Spark environment, eliminating "works on my machine" problems.

2. **Zero Configuration**: The entire Spark cluster setup requires no manual configuration or troubleshooting.

3. **Isolation**: The containerized environment prevents conflicts with other software installed on the host machine.

4. **Production Similarity**: The local cluster mimics a production Spark environment, making development and testing more reliable.

5. **Resource Control**: Memory and CPU allocation for Spark components can be easily modified through environment variables.

6. **Team Onboarding**: New team members can be productive immediately without spending time on complex setup procedures.

## Performance Considerations

The local Spark cluster is designed to balance development convenience with resource efficiency. By default, it creates two worker nodes with modest resource allocations to prevent overwhelming the host machine. For more intensive workloads, the configuration can be adjusted in the `.env` file:

```properties
# Spark Worker 1
SPARK_WORKER1_CORES=2
SPARK_WORKER1_MEMORY=2G

# Spark Worker 2
SPARK_WORKER2_CORES=2
SPARK_WORKER2_MEMORY=2G
```

This allows developers to scale the local cluster according to their machine's capabilities and their project's requirements.

## Monitoring and Debugging

The DevContainer includes access to the Spark web interfaces for monitoring and debugging:

1. **Spark Master UI**: Available at http://localhost:8080
2. **Spark History Server**: Available at http://localhost:18080
3. **Application UI**: Available at http://localhost:4040 (when an application is running)

These interfaces provide valuable insights into job execution, resource usage, and performance bottlenecks, facilitating efficient debugging and optimization.

## Conclusion

The DevContainer-based local Spark cluster offers a streamlined approach to Spark development, eliminating the complexity traditionally associated with setting up and maintaining a local development environment. By combining the containerization benefits of Docker with the development workflow enhancements of DevContainers, this solution enables data engineers to focus on creating and optimizing Spark applications rather than wrestling with infrastructure configuration.

This approach aligns perfectly with modern DevOps practices by treating development environments as code, making them reproducible, shareable, and version-controlled. The result is a more efficient development cycle, faster onboarding, and more reliable testing for Spark applications.

All code for this DevContainer setup is available on GitHub and can be easily adapted to suit specific project requirements.