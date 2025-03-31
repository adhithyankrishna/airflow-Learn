# Apache Airflow Documentation

## What is Airflow

Apache Airflow is an open-source workflow automation tool for authoring, scheduling, and monitoring complex data pipelines. It allows you to define workflows as Directed Acyclic Graphs (DAGs) using Python code.

---

## Why Airflow

- **Scalable** - Can be run on-demand or on containers to handle large workloads.  
- **Flexible** - Python code allows writing complex logic and workflows.  
- **Extensible** - Supports creating custom operators, hooks, and plugins.  
- **Visual Monitoring** - Web UI for tracking DAG progress, errors, and logs.  
- **Community Support** - Large ecosystem of providers for integrations.  

---

## Key Components

### DAG (Directed Acyclic Graph)
- A collection of tasks organized with dependencies and executed in a specific order.  
- Must be acyclic, meaning no task can depend on itself (directly or indirectly).  

### Task
- A single unit of work.  

### Operator
- Defines what actually gets done (Python function, Bash command, SQL query, etc.).  
- Common Operators: `PythonOperator`, `BashOperator`, `EmailOperator`.  

### Scheduler
- Continuously monitors all DAGs and triggers tasks as per their schedules.  
- Determines when tasks are ready to be executed and queues them.  

### Executor
- Handles how tasks are executed.  
- Types: `SequentialExecutor`, `LocalExecutor`, `CeleryExecutor`, `KubernetesExecutor`.  

### Metadata Database
- Stores the state of DAGs, tasks, logs, and history.  
- Default: SQLite (good for testing), but in production, use PostgreSQL or MySQL.  

### Web Server (UI)
- Flask application running with Gunicorn.  
- Visualize DAGs, monitor task progress, trigger DAGs, and view logs.  

---

## Executor

Executors are components responsible for running task instances.  
- When the scheduler picks a DAG, it creates a `TaskInstance` object for the DAG and then pushes the `TaskInstance` to queues (In memory).  

---

## How Executors Work Internally

### Scheduler Pushes Tasks to Executor
- When a DAG is triggered, the Scheduler creates `TaskInstance` objects and sets them to a queued state.  
- The Scheduler places these tasks in a queue (in-memory, Redis, or database, depending on the Executor type).  

### Executor Fetches Tasks from Queue
- The Executor continuously polls the queue for tasks to execute.  
- Based on Executor type, tasks are either run locally, distributed to workers, or deployed as Kubernetes pods.  

### Task Execution
- Once a task is picked up, the Executor updates the task state to running.  
- Executes the task via a Python subprocess, Celery worker, or Kubernetes Pod.  
- On completion or failure, the state is updated in the Metadata Database.  

### State Reporting
- Executors report task states (`success`, `failed`, `queued`, `up_for_retry`, etc.) to the Scheduler via the Metadata Database.  

---

## Detailed Breakdown of Each Executor

### 1. SequentialExecutor
- Executes tasks one at a time.  
- Uses Python’s `subprocess` module for task execution.  
- Tasks are run sequentially, making this Executor unsuitable for production.  
- Defined in `airflow.executors.sequential_executor.SequentialExecutor`.  
- Good for local testing and debugging.  

\`\`\`ini
# In airflow.cfg
executor = SequentialExecutor
\`\`\`

---

### 2. LocalExecutor
- Uses Python’s `multiprocessing` module to run tasks concurrently.  
- Tasks are executed in separate subprocesses.  
- Supports limited parallelism (based on system resources).  
- Defined in `airflow.executors.local_executor.LocalExecutor`.  

\`\`\`ini
# In airflow.cfg
executor = LocalExecutor
parallelism = 32             
dag_concurrency = 16        
\`\`\`

---

### 3. CeleryExecutor
- Uses Celery distributed task queue.  
- Supports distributed execution with multiple workers across different machines.  
- Uses Message Brokers like Redis or RabbitMQ for communication.  
- Defined in `airflow.executors.celery_executor.CeleryExecutor`.  

\`\`\`ini
# In airflow.cfg
executor = CeleryExecutor
broker_url = redis://localhost:6379/0
result_backend = redis://localhost:6379/1
\`\`\`

\`\`\`bash
# Worker Scaling
airflow celery worker -q default -c 4
\`\`\`

---

### 4. KubernetesExecutor
- Designed for dynamic scaling in cloud-native environments.  
- Each task is executed as a separate Kubernetes Pod.  
- Provides complete isolation, scalability, and resource management.  
- Defined in `airflow.executors.kubernetes_executor.KubernetesExecutor`.  

\`\`\`ini
# In airflow.cfg
executor = KubernetesExecutor
\`\`\`

#### Benefits
- Auto-scaling.  
- Isolation (each task runs in its own container).  
- Can specify custom Docker images for tasks.  

---

### 5. DaskExecutor
- Uses a Dask cluster to distribute tasks.  
- Suitable for parallel computation and data-intensive workloads.  
- Requires Dask Scheduler and Workers to be running.  
- Defined in `airflow.executors.dask_executor.DaskExecutor`.  

\`\`\`ini
# In airflow.cfg
executor = DaskExecutor
\`\`\`

---

### 6. DebugExecutor
- Runs tasks synchronously, making it perfect for debugging.  
- Executes tasks in the same process as the Scheduler, eliminating parallelism.  
- Defined in `airflow.executors.debug_executor.DebugExecutor`.  

---

## KubernetesExecutor (Cloud-Native Task Execution)

### Meaning
- Kubernetes is an open-source platform for managing containers (isolated environments where applications run).  
- The `KubernetesExecutor` in Airflow allows you to run each Airflow task as a separate Kubernetes pod (isolated mini-computer).  

---

### How It Works
- **Kubernetes Cluster**: A collection of worker machines (nodes) where pods run.  
- **Pods**: The smallest unit in Kubernetes, each running one or more containers.  
- **Scheduler & API Server**: Handles creating, managing, and deleting pods.  

### How Airflow Uses Kubernetes
- When a task needs to run, Airflow’s Scheduler creates a pod for that task in the Kubernetes cluster.  
- The task runs inside the pod and then the pod is destroyed once it completes.  
- Each task is isolated, making it very secure and efficient.  

### Why It's Powerful
- **Dynamic Scaling**: Tasks can scale up or down based on demand.  
- **Isolation**: Each task runs in its own environment, preventing conflicts.  
- **Resource Efficiency**: Only consumes resources when tasks are running.  
- Ideal for cloud-native, high-scale deployments.  