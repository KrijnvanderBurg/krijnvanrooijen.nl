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

The complete setup with the full code and a condensed explanation is available and linked at the bottom of this article.

## Why a Local Spark DevContainer?
- **Multi-node testing**: Test Data Pipelines on an actual multi-node cluster.
- **Zero setup**: The environment builds automatically with a single command.
- **Consistency**: Every developer works with identical Spark configurations.
- **Isolation**: No conflicts with other installed software.
- **Version control**: Environment definitions are committed to the repository.

### Not familiar with DevContainers?
Previously I wrote a mini-series on DevContainers, how to create, improve, and optimize standardized development environments for your team. Check out the full series below:

- [DevContainers Introduction: The Ideal Standardized Team Development Environment — Part 1/3](https://krijnvanrooijen.nl/blog/decontainers-the-ideal-team-environment/)
- [DevContainers Improved: Integrating Code Quality Checks for Continuous Feedback — Part 2/3](https://krijnvanrooijen.nl/blog/devcontainers-add-code-quality-tools/)
- [DevContainers Mastered: Automating Manual Workflows with VSCode Tasks — Part 3/3](https://krijnvanrooijen.nl/blog/devcontainers-automate-workflow-tasks/)


## DevContainer Cluster Architecture
The local cluster is composed of:

- **DevContainer**: Python, Spark, Jupyter, and CLI tools.
- **Spark Master**: Schedules jobs and manages cluster state.
- **Two Spark Workers**: Simulate distributed compute.
- **Spark History Server**: View and debug completed jobs.

All services run on a private Docker network. The Spark master is reachable at `spark://spark-master:7077` from within the containers. Key UIs are exposed to the host.

![Cluster Diagram](/assets/graphics/2025-04-20-local-spark-cluster-devcontainer/design.drawio.png)

### Networking Considerations in Detail

Spark's distributed architecture creates unique networking challenges in containerized environments. The configuration addresses these through several key strategies:

#### 1. Driver-to-Master Communication
In a standard Spark setup, the driver must communicate bidirectionally with the Spark master and workers. In a containerized environment, this creates a challenge because:

- The driver runs in the DevContainer
- The Spark master and workers run in separate containers
- All containers need to find and communicate with each other

This solution addresses these challenges through these key configurations:

1. **Container Hostnames**: Each container has a defined hostname (`devcontainer`, `spark-master`, etc.) which creates DNS entries in the Docker network.

2. **Driver Host Configuration**: The crucial setting `spark.driver.host=devcontainer` tells Spark to use the DevContainer's hostname for callbacks, ensuring that workers can communicate back to the driver.

3. **Binding to All Interfaces**: Using `spark.driver.bindAddress=0.0.0.0` ensures the driver listens on all network interfaces, not just localhost.

### Accessing the Cluster
The setup exposes Spark’s web UIs for real-time monitoring:

- **Spark Master UI** – [localhost:8080](http://localhost:8080)  
  View cluster status and active applications.
- **Worker UIs** – [localhost:8081](http://localhost:8081), [localhost:8082](http://localhost:8082)  
  Inspect each worker's metrics.
- **History Server** – [localhost:18080](http://localhost:18080)  
  Review finished jobs.
- **Application UI** – [localhost:4040](http://localhost:4040)  
  Track active jobs submitted by spark-submit.

These tools are invaluable for debugging and performance tuning during development.

## Getting Started

1. Download the Spark DevContainer folder from [GitHub](https://github.com/KrijnvanderBurg/.devcontainer/tree/main/spark).
2. Place it inside your project's `.devcontainer/` folder.
3. Open the repo in VSCode with the Dev Containers extension installed.
4. Press `F1` → **Dev Containers: Rebuild and Reopen in Container**

That’s it. A full multi-node Spark cluster is now running locally inside your DevContainer.

### Running PySpark Jobs
You can run full PySpark jobs inside the DevContainer using either:

Just hit **Ctrl+Shift+B** This will trigger a predefined task that runs the following command:

```bash
/opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --conf spark.driver.host=devcontainer \
  --conf spark.driver.bindAddress=0.0.0.0 \
  your_script.py
```

### Using Jupyter Notebooks in VSCode
This setup supports native `.ipynb` execution inside VSCode — no browser, no token, no hassle.

Just open a notebook file. The interpreter auto-connects to PySpark and launches by preloading the following `SparkSession` in the background. This can be found in the `spark-init.py` file which is added to Jupyter environment.

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

## Implementation Details

The power of this solution lies in its detailed configuration. Let's explore each component and understand why specific choices were made.

### DevContainer Configuration: The Developer Experience
The DevContainer is the primary environment where developers work. Its configuration focuses on creating a seamless experience with all the necessary tools pre-configured.

```json
{
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-toolsai.jupyter",
                "ms-toolsai.datawrangler",
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
                // Task for submitting Spark jobs
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

### DevContainer Dockerfile
The DevContainer is created via a Dockerfile in the Docker-Compose. It installs Java, Spark, Pyspark, Jupyter, and more by default. Meaning, everything is included in the image and does not require to be installed by the user anymore. The added benefit is that all team members are running the exact same environment.

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

### Docker Compose Configuration: Orchestrating the Cluster
The Docker Compose file is the heart of this solution. It orchestrates multiple containers to create a complete Spark cluster environment.

This configuration includes several advanced Docker Compose features:

1. **YAML Anchors and Templates**: The `x-spark-worker-template` defines a reusable template for worker nodes, reducing duplication and ensuring consistency. This follows the DRY (Don't Repeat Yourself) principle.
2. **Environment Variable Substitution**: Variables like `${SPARK_VERSION}` are populated from the `.env` file, making the configuration highly customizable without editing the compose file.
3. **Volume Management**: The setup uses named volumes for logs and events, ensuring data persistence between container restarts. It also mounts the project directory to `/workspace` in all containers.
4. **Network Isolation**: A dedicated `spark-network` connects all containers, providing isolation and consistent name resolution.
5. **Port Mapping**: All necessary UI and communication ports are explicitly mapped to make them accessible from the host machine. It is a careful balance of connectivity between the docker containers while changing the URLs via `SPARK_PUBLIC_DNS` to set the correct URLs in the Spark web UI.

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

### Environment Configuration: Centralized Control
The `.env` file provides a single location to customize the entire environment. It is automatically loaded by docker-compose.

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

🔗 [GitHub – .devcontainer Spark Multi-Node Cluster Development Environment](https://github.com/KrijnvanderBurg/.devcontainer/tree/main/spark)
