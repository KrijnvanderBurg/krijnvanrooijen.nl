---
title: "[DRAFT] Local Spark Cluster DevContainer: Streamline Data Engineering Workflows"
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

This article presents a solution that creates a complete, production-like Spark cluster locally using a DevContainer with Docker Compose. The setup includes a Spark master, multiple worker nodes, history server, Jupyter server — all in a development environment with PySpark and Jupyter notebooks integration — configured automatically and ready to use.

## Why a Local Spark DevContainer?

Setting up a local Spark development environment traditionally involves:

1. Installing compatible versions of Java, Spark, and Python
2. Configuring environment variables and network settings
3. Managing dependencies between different components
4. Troubleshooting inconsistent behavior across machines

By containerizing the entire environment, these challenges are eliminated by automation. The DevContainer approach provides:

- **Consistency**: Every developer works with identical Spark configurations
- **Zero setup**: The environment builds automatically with a single command
- **Isolation**: No conflicts with other installed software
- **Version control**: Environment definitions are committed to the repository
- **Multi-node testing**: Test distributed operations on multiple worker nodes

### Not familiar with DevContainers?

Previously I wrote a mini-series on DevContainers, how to create, improve, and optimize standardized development environments for your team. Check out the full series below:

- [DevContainers Introduction: The Ideal Standardized Team Development Environment — Part 1/3](https://krijnvanrooijen.nl/blog/decontainers-the-ideal-team-environment/)
- [DevContainers Improved: Integrating Code Quality Checks for Continuous Feedback — Part 2/3](https://krijnvanrooijen.nl/blog/devcontainers-add-code-quality-tools/)
- [DevContainers Mastered: Automating Manual Workflows with VSCode Tasks — Part 3/3](https://krijnvanrooijen.nl/blog/devcontainers-automate-workflow-tasks/)

## DevContainer Architecture

The solution consists of five interconnected Docker containers orchestrated through Docker Compose:

![Spark Cluster Architecture](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/design.drawio.png)

1. **DevContainer**: Development environment with Java, Spark, PySpark, and Jupyter pre-installed.
2. **Spark Master**: Central cluster coordinator.
3. **Spark Workers (1-2)**: Nodes that execute distributed tasks in the cluster.
4. **History Server**: Tracks completed jobs for inspection and debugging.

All containers communicate through a shared Docker bridge network to ensure seamless communication without needing to expose internal ports unnecessarily.

## Getting Started

To use this DevContainer:

1. Download the Spark DevContainer folder from my [GitHub](https://github.com/KrijnvanderBurg/.devcontainer/tree/main/spark).
2. Place the Spark folder inside the `.devcontainer` folder in your project root.
3. Open the folder in VSCode with the DevContainers extension installed.
   - Press `F1`, then select **Dev Containers: Rebuild and Reopen in Container**
   - The setup will auto-build the environment and start up all containers.

That's it — you're inside a full Spark cluster running locally.

### PySpark Jobs

A major benefit of this setup is the ability to run full-fledged PySpark scripts directly against the local Spark cluster. Here's how it works:

✅ How to Run PySpark Jobs

Once you're inside the DevContainer, you can execute any .py file using spark-submit. There are two primary ways to do it:

**Option 1: VSCode Task (Recommended)**

The DevContainer includes a VSCode tasks.json entry that lets you run the current file as a Spark job with a simple shortcut.

Usage:
1. Open a .py file with your PySpark code.
2. Press Ctrl+Shift+B (Windows/Linux) or Cmd+Shift+B (Mac) to trigger the default build task.
3. VSCode will use spark-submit.sh to submit your script to the cluster.

Under the hood, it runs:

```bash
/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --conf spark.driver.host=devcontainer \
  --conf spark.driver.bindAddress=0.0.0.0 \
  your_script.py
```
This allows for consistent submission no matter the job or file structure.

**Option 2: Manual Script Execution**
Prefer running things manually? You can do that too.

Usage:
```bash
.devcontainer/spark/spark-submit.sh path/to/your_script.py
```

This gives you full control over your job execution, handy when you're debugging or customizing configurations.


### Jupyter Notebook Support (VSCode-Only, No Browser)

This DevContainer setup fully supports running Jupyter notebooks directly within VSCode — no external browser, no token auth, no Jupyter server process. Everything executes natively inside the container.

How It Works

VSCode detects .ipynb files and launches the built-in notebook interface. The kernel automatically connects to the Python interpreter inside the container, which comes preconfigured with pyspark.

The container also auto-loads a Spark session on startup, so you're ready to go without any manual setup.

#### Preloaded Spark Context

The container initializes Spark automatically using an IPython startup script.

That script runs the following:
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("VSCode Notebook") \
    .master("spark://spark-master:7077") \
    .config("spark.driver.host", "devcontainer") \
    .config("spark.driver.bindAddress", "0.0.0.0") \
    .config("spark.ui.port", "4040") \
    .getOrCreate()

sc = spark.sparkContext
sc.setLogLevel("WARN")
```

This means every notebook session starts with spark and sc pre-initialized.


## Implementation Details

### DevContainer Configuration

The DevContainer includes essential extensions and VSCode settings for Spark and Jupyter development:


## Implementation Details

### DevContainer configuration

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
                "jupyter.kernelSpecConnectionMetadata": [
                    {
                        "kernelspec": {
                            "name": "pyspark",
                            "display_name": "PySpark"
                        }
                    }
                ],
                
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
                            },
                            "presentation": {
                                "reveal": "always",
                                "panel": "new"
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

```Dockerfile
ARG PYTHON_VERSION=3.10
FROM python:${PYTHON_VERSION}-bullseye

# Install Java and other dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    openjdk-11-jdk \
    wget \
    procps \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

# Arguments for Spark setup
ARG SPARK_VERSION=3.5.5
ARG HADOOP_VERSION=3

# Set Spark environment variables
ENV SPARK_VERSION=${SPARK_VERSION}
ENV HADOOP_VERSION=${HADOOP_VERSION}
ENV SPARK_HOME=/opt/spark
ENV PATH=$PATH:$SPARK_HOME/bin

# Download and install Spark
RUN wget -q https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz && \
    tar -xzf spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz && \
    mv spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION} /opt/spark && \
    rm spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz

# Install Python dependencies including Jupyter
RUN pip install --no-cache-dir \
    pyspark==${SPARK_VERSION} \
    jupyter \
    jupyterlab \
    ipykernel 

# Set up a Jupyter kernel with auto Spark initialization
RUN python -m ipykernel install --name pyspark --display-name "PySpark" && \
    mkdir -p /root/.ipython/profile_default/startup/

# Create startup script for auto Spark initialization
COPY spark_init.py /root/.ipython/profile_default/startup/00-spark-init.py

# Create a Jupyter kernel for PySpark
RUN ipython kernel install --user --name=pyspark --display-name='PySpark'

WORKDIR /workspace

CMD ["sleep", "infinity"]
```


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
    build:
      context: .
      dockerfile: ./Dockerfile
      args:
        - SPARK_VERSION=${SPARK_VERSION}
        - HADOOP_VERSION=3
        - PYTHON_VERSION=${PYTHON_VERSION}
    volumes:
      - ../../:/workspace
    networks:
      - spark-network
    ports:
      - "4040:4040"  # Expose Spark application UI
    environment:
      - SPARK_LOCAL_IP=0.0.0.0
      - SPARK_DRIVER_HOST=devcontainer
      - SPARK_PUBLIC_DNS=localhost
      - SPARK_MASTER_URL=spark://spark-master:${SPARK_MASTER_PORT}
  
  # Spark master node
  spark-master:
    image: apache/spark:${SPARK_VERSION}
    command: "/opt/spark/bin/spark-class org.apache.spark.deploy.master.Master"
    hostname: spark-master
    environment:
      - SPARK_MASTER_HOST=spark-master
      - SPARK_MASTER_PORT=${SPARK_MASTER_PORT}
      - SPARK_MASTER_WEBUI_PORT=${SPARK_MASTER_WEBUI_PORT}
      - SPARK_PUBLIC_DNS=localhost  # For UI URLs to be accessible from host
      - SPARK_LOCAL_IP=spark-master  # Keep internal hostname for Docker network communication
    ports:
      - "${SPARK_MASTER_WEBUI_PORT}:${SPARK_MASTER_WEBUI_PORT}"   # Web UI
      - "${SPARK_MASTER_PORT}:${SPARK_MASTER_PORT}"   # Spark master port
    volumes:
      - spark-logs:/opt/spark/logs
      - spark-events:/opt/spark/events
      - ./spark-defaults.conf:/opt/spark/conf/spark-defaults.conf
      - ../../:/workspace
    networks:
      - spark-network
    depends_on:
      - devcontainer
    restart: unless-stopped
  
  # Worker nodes using the template
  spark-worker-1:
    <<: *spark-worker-template
    hostname: spark-worker-1
    environment:
      - SPARK_WORKER_CORES=${SPARK_WORKER1_CORES}
      - SPARK_WORKER_MEMORY=${SPARK_WORKER1_MEMORY}
      - SPARK_WORKER_WEBUI_PORT=${SPARK_WORKER1_WEBUI_PORT}
      - SPARK_PUBLIC_DNS=localhost  # For UI URLs to be accessible from host
      - SPARK_LOCAL_IP=spark-worker-1  # Keep internal hostname for Docker network communication
    ports:
      - "${SPARK_WORKER1_WEBUI_PORT}:${SPARK_WORKER1_WEBUI_PORT}"   # Worker 1 UI

  spark-worker-2:
    <<: *spark-worker-template
    hostname: spark-worker-2
    environment:
      - SPARK_WORKER_CORES=${SPARK_WORKER2_CORES}
      - SPARK_WORKER_MEMORY=${SPARK_WORKER2_MEMORY}
      - SPARK_WORKER_WEBUI_PORT=${SPARK_WORKER2_WEBUI_PORT}
      - SPARK_PUBLIC_DNS=localhost  # For UI URLs to be accessible from host
      - SPARK_LOCAL_IP=spark-worker-2  # Keep internal hostname for Docker network communication
    ports:
      - "${SPARK_WORKER2_WEBUI_PORT}:${SPARK_WORKER2_WEBUI_PORT}"   # Worker 2 UI

  # History server has no dependencies - keeps running independently
  spark-history-server:
    image: apache/spark:${SPARK_VERSION}
    command: "/opt/spark/bin/spark-class org.apache.spark.deploy.history.HistoryServer"
    hostname: spark-history
    environment:
      - SPARK_PUBLIC_DNS=localhost
      - SPARK_LOCAL_IP=0.0.0.0
    ports:
      - "${SPARK_HISTORY_WEBUI_PORT}:${SPARK_HISTORY_WEBUI_PORT}"   # History UI
    volumes:
      - spark-logs:/opt/spark/logs
      - spark-events:/opt/spark/events
      - ./spark-defaults.conf:/opt/spark/conf/spark-defaults.conf
    networks:
      - spark-network
    restart: unless-stopped

networks:
  spark-network:
    driver: bridge

volumes:
  spark-logs:
  spark-events:
```

spark submit shell script
```bash
#!/bin/bash
FILE_TO_SUBMIT=${1}

/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --conf spark.driver.host=devcontainer \
  --conf spark.driver.bindAddress=0.0.0.0 \
  "$FILE_TO_SUBMIT"
```

### Environment Configuration
The `.env` file provides centralized configuration of Spark versions, Python versions, and resource allocation, which is automatically used by docker-compose.

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
# Use first argument as the file to submit, or default to sample_job.py if no argument provided
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
   ![Spark Master UI](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/spark-master-ui.png)

2. **Spark History Server** (http://localhost:18080): Examine completed jobs and their performance metrics
   ![Spark History Server](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/spark-history-server.png)

3. **Application UI** (http://localhost:4040): Monitor running applications in real-time
   ![Application UI](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/spark-application-ui.png)

These interfaces provide valuable insights for debugging and optimization during development.


## Extending the DevContainer
The base DevContainer can be extended for specific use cases:

1. **Additional Libraries**: Add Python packages to the Dockerfile
2. **Custom Configurations**: Mount additional configuration files into the containers
3. **Integration with Other Services**: Add services like PostgreSQL or Kafka to the Docker Compose file

For example, to add PostgreSQL for storing processed data:

```yaml
# Add to docker-compose.yml
postgres:
  image: postgres:14
  environment:
    - POSTGRES_PASSWORD=postgres
    - POSTGRES_USER=postgres
    - POSTGRES_DB=sparkdata
  ports:
    - "5432:5432"
  volumes:
    - postgres-data:/var/lib/postgresql/data
  networks:
    - spark-network
```

## Conclusion

The Spark cluster DevContainer provides a comprehensive, production-like environment for local development with minimal setup. By containerizing the entire Spark stack, it eliminates configuration headaches and ensures consistency across team members.

This approach aligns with modern DevOps practices by treating development environments as code, making them reproducible, shareable, and version-controlled. The result is a more efficient development cycle and more reliable testing for Spark applications.

All code for this DevContainer setup is available on GitHub and can be adapted to specific project requirements.
