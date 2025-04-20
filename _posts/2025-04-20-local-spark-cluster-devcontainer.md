```markdown
// filepath: /home/x0r/Documents/GitHub/krijnvanrooijen.nl/_posts/2025-04-20-local-spark-cluster-devcontainer.md
---
title: "Local Spark Cluster DevContainer: Streamline Data Engineering Workflows"
date: 2025-04-20
excerpt: "Create a fully functional, containerized Spark cluster development environment with minimal setup using DevContainers and Docker Compose."
tags:
- DevContainer
- Data Engineering
- PySpark
- Docker
- Spark
image: /assets/graphics/2025-04-20-local-spark-cluster-devcontainer/thumbnail.jpg
pin: false
---

Data engineering workflows with Apache Spark often involve complex setup procedures, environment configuration, and dependency management. Traditional approaches to creating local Spark development environments frequently lead to "works on my machine" problems, inconsistent behavior across team members' setups, and significant time spent on configuration rather than actual development.

This article presents a DevContainer solution that creates a complete, production-like Spark cluster locally using Docker Compose. The setup includes a Spark master, multiple worker nodes, history server, and development environment with PySpark and Jupyter notebooks integration - all configured automatically and ready to use.

## Why a Local Spark DevContainer?

Setting up a local Spark development environment traditionally involves:

1. Installing compatible versions of Java, Spark, and Python
2. Configuring environment variables and network settings
3. Managing dependencies between different components
4. Troubleshooting inconsistent behavior across machines

By containerizing the entire environment, these challenges are eliminated. The DevContainer approach provides:

- **Consistency**: Every developer works with identical Spark configurations
- **Zero setup**: The environment builds automatically with a single command
- **Isolation**: No conflicts with other installed software
- **Version control**: Environment definitions are committed to the repository
- **Multi-node testing**: Test distributed operations on multiple worker nodes

## DevContainer Architecture

The solution consists of five interconnected Docker containers orchestrated through Docker Compose:

![Spark Cluster Architecture](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/architecture-diagram.png)

1. **DevContainer**: Primary development environment with PySpark, Jupyter, and VSCode integration
2. **Spark Master**: Coordination node that manages the cluster resources
3. **Spark Workers (1-2)**: Execution nodes that perform the actual computation
4. **History Server**: Web interface for viewing completed Spark job metrics and logs

All containers are connected through a dedicated Docker network, allowing seamless communication between components while isolating them from the host network.

## Getting Started

To use this DevContainer:

1. Clone the repository containing the DevContainer configuration
2. Open in VSCode with the DevContainers extension installed
3. When prompted, select "Reopen in Container"

The entire environment will build automatically, with no additional configuration required.

## Key Implementation Details

### Docker Compose Configuration

The Docker Compose file uses YAML anchors and templates to create a maintainable configuration with multiple worker nodes:

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
  # Development environment with all tools installed
  devcontainer:
    # ...existing code...
  
  # Spark master node
  spark-master:
    # ...existing code...
  
  # Worker nodes using the template
  spark-worker-1:
    <<: *spark-worker-template
    hostname: spark-worker-1
    environment:
      - SPARK_WORKER_CORES=${SPARK_WORKER1_CORES}
      - SPARK_WORKER_MEMORY=${SPARK_WORKER1_MEMORY}
      # ...existing code...

  spark-worker-2:
    <<: *spark-worker-template
    hostname: spark-worker-2
    # ...existing code...
```

### Environment Configuration

The `.env` file provides centralized configuration of Spark versions, Python versions, and resource allocation:

```properties
# Spark configuration
SPARK_VERSION=3.5.5
PYTHON_VERSION=3.11

# Spark Master
SPARK_MASTER_WEBUI_PORT=8080
SPARK_MASTER_PORT=7077

# Spark Worker 1
SPARK_WORKER1_CORES=2
SPARK_WORKER1_MEMORY=2G
SPARK_WORKER1_WEBUI_PORT=8081

# Spark Worker 2
SPARK_WORKER2_CORES=2
SPARK_WORKER2_MEMORY=2G
SPARK_WORKER2_WEBUI_PORT=8082

# Spark History Server
SPARK_HISTORY_WEBUI_PORT=18080
```

This approach allows easy adjustment of resource allocation and versions without modifying the core Docker Compose configuration.

### VSCode Integration

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
                // Jupyter notebook integration
                "jupyter.interactiveWindow.textEditor.executeSelection": true,
                "jupyter.notebookFileRoot": "${workspaceFolder}",
                // ...existing code...
                
                // Tasks for submitting Spark jobs
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

This configuration enables:

1. Automatic notebook initialization with proper Spark configuration
2. One-click submission of PySpark scripts to the cluster
3. Integrated data exploration and visualization tools

## Spark Session Initialization

The environment includes an auto-initialization script for Jupyter notebooks that creates a properly configured Spark session:

```python
# Initialize Spark with optimal settings for development
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("Jupyter Notebook") \
    .master("spark://spark-master:7077") \
    .config("spark.driver.host", "devcontainer") \
    .config("spark.driver.bindAddress", "0.0.0.0") \
    .config("spark.ui.port", "4040") \
    .config("spark.dynamicAllocation.enabled", "false") \
    .config("spark.scheduler.mode", "FAIR") \
    .config("spark.python.worker.reuse", "true") \
    .getOrCreate()

# SparkContext for backward compatibility
sc = spark.sparkContext

# Set log level to reduce noise
sc.setLogLevel("WARN")
```

This script handles the complex network configuration required for the containerized environment, ensuring that the Spark driver (running in the DevContainer) can communicate properly with the Spark master and workers.

## Networking Considerations

One of the most challenging aspects of running Spark in containers is networking. The DevContainer configuration addresses several common issues:

1. **Driver-to-Master Communication**: The DevContainer's hostname is used as the driver host, allowing the master to communicate back to the driver.

2. **External Access**: All UI ports are mapped to the host machine, enabling access to Spark UIs from the host browser.

3. **Inter-node Communication**: Containers use their service names as hostnames within the Docker network.

This configuration ensures that all components can communicate effectively without manual network setup.

## Working with the Spark Cluster

### Submitting PySpark Jobs

The environment includes a convenience script for submitting PySpark jobs to the cluster:

```bash
#!/bin/bash
# Use first argument as the file to submit
FILE_TO_SUBMIT=${1}

/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --conf spark.driver.host=devcontainer \
  --conf spark.driver.bindAddress=0.0.0.0 \
  "$FILE_TO_SUBMIT"
```

This script is integrated into VSCode tasks, allowing submission of the current file with the keyboard shortcut `Ctrl+Shift+B`.

### Monitoring Jobs

The DevContainer setup provides three web interfaces for monitoring:

1. **Spark Master UI** (http://localhost:8080): View cluster status, worker nodes, and running applications
2. **Spark History Server** (http://localhost:18080): Examine completed jobs and their performance metrics
3. **Application UI** (http://localhost:4040): Monitor running applications in real-time

These interfaces provide valuable insights for debugging and optimization during development.

### Example Workflow

A typical development workflow with this environment:

1. Open a Jupyter notebook for interactive exploration
2. Develop and test transformations interactively
3. Move finalized code to a PySpark script
4. Submit the script to the cluster using the VSCode task
5. Monitor execution using the Spark UIs
6. View history and logs in the History Server after completion

This workflow provides a seamless experience from exploration to execution.

## Performance Tuning

The local Spark cluster is configured with sensible defaults for development, but can be tuned for specific workloads:

1. **Worker Resources**: Adjust cores and memory in the `.env` file based on the host machine capabilities
2. **Executor Settings**: Configure executor memory and cores in the notebook initialization or submit script
3. **Parallelism**: Set the default parallelism based on the available worker cores

For memory-intensive workloads, consider:

```python
spark = SparkSession.builder \
    .config("spark.executor.memory", "1g") \
    .config("spark.driver.memory", "2g") \
    .config("spark.memory.offHeap.enabled", "true") \
    .config("spark.memory.offHeap.size", "1g") \
    # ...existing code...
    .getOrCreate()
```

## Common Challenges and Solutions

### Connection Refused Errors

If Spark jobs fail with "connection refused" errors, verify that:

1. The driver host is correctly set to `devcontainer`
2. The driver bind address is set to `0.0.0.0`
3. The master URL uses the service name: `spark://spark-master:7077`

### Memory Issues

For "out of memory" errors:

1. Reduce the worker memory allocation in the `.env` file
2. Add explicit garbage collection hints in your code
3. Consider enabling off-heap memory

### Slow Performance

To improve performance:

1. Use appropriate partitioning for your data
2. Enable caching for frequently accessed DataFrames
3. Adjust the number of shuffle partitions based on data size

## Extending the DevContainer

The base DevContainer can be extended for specific use cases:

1. **Additional Libraries**: Add Python packages to the Dockerfile
2. **Custom Configurations**: Mount additional configuration files into the containers
3. **Integration with Other Services**: Add services like PostgreSQL or Kafka to the Docker Compose file

## Conclusion

The Spark cluster DevContainer provides a comprehensive, production-like environment for local development with minimal setup. By containerizing the entire Spark stack, it eliminates configuration headaches and ensures consistency across team members.

This approach aligns with modern DevOps practices by treating development environments as code, making them reproducible, shareable, and version-controlled. The result is a more efficient development cycle and more reliable testing for Spark applications.

All code for this DevContainer setup is available on GitHub and can be adapted to specific project requirements.
```