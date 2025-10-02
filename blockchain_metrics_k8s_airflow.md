# Blockchain Metrics Framework: Airflow on Kubernetes Architecture

## Overview

This document outlines the architecture for a scalable blockchain metrics framework using Apache Airflow deployed on Kubernetes. The system leverages the KubernetesPodOperator to dynamically spawn pods for metric calculations, providing improved isolation, resource management, and scalability. The Lambda architecture processes data from both real-time blockchain streams and historical PostgreSQL time-series databases, using Redis as a durable buffer layer.

## Airflow on Kubernetes Architecture

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │                 │    │         Airflow Pods            │ │
│  │  Redis Buffer   │    │                                 │ │
│  │                 │    │  ┌─────────┐    ┌─────────────┐ │ │
│  └─────────────────┘    │  │Webserver│    │ Scheduler   │ │ │
│          │              │  └─────────┘    └─────────────┘ │ │
│          │              │                                 │ │
│          │              │  ┌─────────┐    ┌─────────────┐ │ │
│          └──────────────┼─▶│ Worker  │    │PostgreSQL DB│ │ │
│                         │  └─────────┘    └─────────────┘ │ │
│                         │                        │        │ │
│                         └────────────────────────┼────────┘ │
│                                                  │          │
│  ┌─────────────────────────────────────┐         │          │
│  │       Dynamic Task Pods              │         │          │
│  │                                     │◀────────┘          │
│  │  ┌─────────┐ ┌─────────┐ ┌────────┐ │                    │
│  │  │Metric A │ │Metric B │ │Metric C│ │                    │
│  │  │  Pod    │ │  Pod    │ │  Pod   │ │                    │
│  │  └─────────┘ └─────────┘ └────────┘ │                    │
│  │                                     │                    │
│  └─────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

1. **Airflow Core Components**:
   - **Webserver**: UI for DAG monitoring and management (deployed as a Kubernetes Deployment)
   - **Scheduler**: Triggers task execution (deployed as a StatefulSet for reliability)
   - **Workers**: Executes tasks that don't require KubernetesPodOperator (optional)
   - **PostgreSQL DB**: Stores Airflow metadata (deployed as a StatefulSet with persistent storage)

2. **Dynamic Task Pods**:
   - Spawned on-demand by KubernetesPodOperator
   - Isolated environment for each metric calculation
   - Auto-scaling based on workload
   - Resource limits and requests for optimal cluster utilization

3. **Redis Buffer**:
   - Deployed as a StatefulSet with persistent storage
   - Configured with high availability using Redis Sentinel
   - Optimized for durability with AOF persistence

## KubernetesPodOperator Implementation

The KubernetesPodOperator allows us to dynamically create pods for each task execution, providing several advantages:

- **Isolation**: Each metric calculation runs in its own container
- **Resource Management**: Specific CPU/memory allocation per metric
- **Scalability**: Kubernetes handles pod scheduling and scaling
- **Flexibility**: Custom container images for different metric types

### DAG Implementation with KubernetesPodOperator

```python
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from kubernetes.client import models as k8s
from datetime import datetime, timedelta

# Define resource requirements
resource_requirements = {
    "small": {"request_memory": "256Mi", "request_cpu": "100m", "limit_memory": "512Mi", "limit_cpu": "200m"},
    "medium": {"request_memory": "512Mi", "request_cpu": "500m", "limit_memory": "1Gi", "limit_cpu": "1000m"},
    "large": {"request_memory": "1Gi", "request_cpu": "1000m", "limit_memory": "2Gi", "limit_cpu": "2000m"},
}

def create_metric_dag(
    metric_name,
    metric_type,
    bucket_size,
    schedule_interval,
    resources="medium",
    image="metrics-calculator:latest",
    namespace="blockchain-metrics"
):
    """Factory function to create a DAG for a specific metric."""
    
    dag_id = f"metric_{metric_name}_{bucket_size}"
    
    default_args = {
        'owner': 'airflow',
        'depends_on_past': False,
        'start_date': datetime(2023, 1, 1),
        'email_on_failure': True,
        'email_on_retry': False,
        'retries': 3,
        'retry_delay': timedelta(minutes=5),
    }
    
    # Configure resources based on metric needs
    res = resource_requirements[resources]
    
    with DAG(
        dag_id,
        default_args=default_args,
        schedule_interval=schedule_interval,
        catchup=False,
        max_active_runs=1,
        tags=['metrics', metric_name, bucket_size],
    ) as dag:
        
        # Define volume mounts for configuration and shared data
        volume_mounts = [
            k8s.V1VolumeMount(
                name="metrics-config", 
                mount_path="/opt/metrics/config",
                read_only=True
            ),
        ]
        
        # Define volumes
        volumes = [
            k8s.V1Volume(
                name="metrics-config",
                config_map=k8s.V1ConfigMapVolumeSource(
                    name=f"metrics-config-{metric_name}"
                )
            ),
        ]
        
        # Create calculation task using KubernetesPodOperator
        calculate_metric = KubernetesPodOperator(
            task_id=f"calculate_{metric_name}_{bucket_size}",
            name=f"calculate-{metric_name}-{bucket_size}",
            namespace=namespace,
            image=image,
            cmds=["python", "-m", "metrics.calculator"],
            arguments=[
                "--metric-name", metric_name,
                "--metric-type", metric_type,
                "--bucket-size", bucket_size,
                "--execution-date", "{{ ds }}"
            ],
            # Resource specifications
            resources=k8s.V1ResourceRequirements(
                requests={
                    "memory": res["request_memory"],
                    "cpu": res["request_cpu"]
                },
                limits={
                    "memory": res["limit_memory"],
                    "cpu": res["limit_cpu"]
                }
            ),
            # Kubernetes specifics
            volumes=volumes,
            volume_mounts=volume_mounts,
            is_delete_operator_pod=True,
            get_logs=True,
            startup_timeout_seconds=300,
            # Labels for organizational purposes
            labels={
                "app": "blockchain-metrics",
                "metric": metric_name,
                "bucket": bucket_size
            },
            # Environment variables
            env_vars={
                "POSTGRES_HOST": "{{ var.value.postgres_host }}",
                "POSTGRES_PORT": "{{ var.value.postgres_port }}",
                "POSTGRES_DB": "{{ var.value.postgres_db }}",
                "POSTGRES_USER": "{{ var.value.postgres_user }}",
                "POSTGRES_PASSWORD": "{{ var.value.postgres_password }}",
                "REDIS_HOST": "{{ var.value.redis_host }}",
                "REDIS_PORT": "{{ var.value.redis_port }}"
            }
        )
        
        # Add additional tasks as needed (validation, notification, etc.)
        
        return dag
```

## Dynamic DAG Builder for Kubernetes

The DAG Builder pattern is enhanced to work with Kubernetes:

```python
class KubernetesMetricDagFactory:
    """Factory for creating metric DAGs using KubernetesPodOperator."""
    
    def __init__(self, metric_registry_client, config_manager):
        self.metric_registry = metric_registry_client
        self.config_manager = config_manager
        self.dag_directory = "/opt/airflow/dags/generated/"
        
    def generate_all_dags(self):
        """Generate DAGs for all metrics in the registry."""
        metrics = self.metric_registry.get_all_metrics()
        
        for metric in metrics:
            # For each metric, create the necessary ConfigMaps
            self._create_metric_configmap(metric)
            
            # Generate DAGs for each time bucket
            self._generate_metric_dags(metric)
    
    def _create_metric_configmap(self, metric):
        """Create or update Kubernetes ConfigMap for metric configuration."""
        config_data = {
            "metric.json": json.dumps(metric.to_dict()),
            "calculation_logic.sql": metric.calculation_logic.get("sql_template", ""),
            "dependencies.json": json.dumps(metric.dependencies)
        }
        
        self.config_manager.create_or_update_configmap(
            name=f"metrics-config-{metric.name}",
            namespace="blockchain-metrics",
            data=config_data
        )
    
    def _generate_metric_dags(self, metric):
        """Generate DAGs for a specific metric across time buckets."""
        for bucket_size in metric.bucket_sizes:
            dag_id = f"metric_{metric.name}_{bucket_size}"
            
            # Determine schedule interval based on bucket size
            schedule_interval = self._get_schedule_for_bucket(bucket_size)
            
            # Determine resource requirements based on metric complexity
            resources = self._get_resources_for_metric(metric)
            
            # Determine appropriate container image
            image = self._get_image_for_metric_type(metric.metric_type)
            
            # Generate DAG file content
            dag_content = f"""
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from kubernetes.client import models as k8s
from datetime import datetime, timedelta

# Import the factory function
from metric_dag_factory import create_metric_dag

# Create the DAG
dag = create_metric_dag(
    metric_name="{metric.name}",
    metric_type="{metric.metric_type}",
    bucket_size="{bucket_size}",
    schedule_interval="{schedule_interval}",
    resources="{resources}",
    image="{image}",
    namespace="blockchain-metrics"
)
"""
            
            # Write DAG file
            with open(f"{self.dag_directory}/{dag_id}.py", "w") as f:
                f.write(dag_content)
    
    def _get_schedule_for_bucket(self, bucket_size):
        """Return appropriate schedule interval for time bucket."""
        if bucket_size == "hourly":
            return "@hourly"
        elif bucket_size == "daily":
            return "@daily"
        elif bucket_size == "weekly":
            return "@weekly"
        else:
            return None
    
    def _get_resources_for_metric(self, metric):
        """Determine resource requirements based on metric complexity."""
        complexity = metric.get("complexity", "medium")
        return complexity  # "small", "medium", or "large"
    
    def _get_image_for_metric_type(self, metric_type):
        """Determine appropriate container image based on metric type."""
        image_mapping = {
            "counter": "metrics-calculator:counter-latest",
            "gauge": "metrics-calculator:gauge-latest",
            "histogram": "metrics-calculator:histogram-latest",
            "default": "metrics-calculator:latest"
        }
        
        return image_mapping.get(metric_type, image_mapping["default"])
```

## Container Implementation

Each metric calculation runs in its own container, providing isolation and scalability. We create specialized container images for different metric types:

### Base Container Image

```dockerfile
FROM python:3.10-slim

# Install dependencies
RUN pip install --no-cache-dir \
    psycopg2-binary \
    redis \
    pandas \
    numpy \
    requests \
    pydantic

# Set up working directory
WORKDIR /opt/metrics

# Copy application code
COPY ./src /opt/metrics/src
COPY ./config /opt/metrics/config

# Set up entrypoint
ENTRYPOINT ["python", "-m", "metrics.calculator"]
```

### Metric Calculator Implementation

```python
# metrics/calculator.py
import argparse
import json
import logging
import os
import sys
from datetime import datetime, timedelta

import pandas as pd
import psycopg2
import redis

# Set up logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger("metrics-calculator")

def parse_args():
    """Parse command-line arguments."""
    parser = argparse.ArgumentParser(description="Calculate blockchain metrics")
    parser.add_argument("--metric-name", required=True, help="Name of the metric to calculate")
    parser.add_argument("--metric-type", required=True, help="Type of metric (counter, gauge, histogram)")
    parser.add_argument("--bucket-size", required=True, help="Size of time bucket (hourly, daily, weekly)")
    parser.add_argument("--execution-date", required=True, help="Execution date in YYYY-MM-DD format")
    return parser.parse_args()

def get_db_connection():
    """Get PostgreSQL database connection."""
    return psycopg2.connect(
        host=os.environ.get("POSTGRES_HOST", "localhost"),
        port=os.environ.get("POSTGRES_PORT", "5432"),
        dbname=os.environ.get("POSTGRES_DB", "blockchain_metrics"),
        user=os.environ.get("POSTGRES_USER", "postgres"),
        password=os.environ.get("POSTGRES_PASSWORD", "postgres")
    )

def get_redis_connection():
    """Get Redis connection."""
    return redis.Redis(
        host=os.environ.get("REDIS_HOST", "localhost"),
        port=int(os.environ.get("REDIS_PORT", "6379")),
        db=0
    )

def load_metric_config(metric_name):
    """Load metric configuration from mounted ConfigMap."""
    try:
        with open(f"/opt/metrics/config/metric.json", "r") as f:
            return json.load(f)
    except FileNotFoundError:
        logger.error(f"Metric configuration file not found for {metric_name}")
        raise

def calculate_time_window(execution_date, bucket_size):
    """Calculate start and end times for the given bucket."""
    dt = datetime.strptime(execution_date, "%Y-%m-%d")
    
    if bucket_size == "hourly":
        start_time = dt.replace(minute=0, second=0, microsecond=0)
        end_time = start_time + timedelta(hours=1)
    elif bucket_size == "daily":
        start_time = dt.replace(hour=0, minute=0, second=0, microsecond=0)
        end_time = start_time + timedelta(days=1)
    elif bucket_size == "weekly":
        # Assuming weeks start on Monday (0 = Monday in Python's weekday())
        weekday = dt.weekday()
        start_time = (dt - timedelta(days=weekday)).replace(hour=0, minute=0, second=0, microsecond=0)
        end_time = start_time + timedelta(days=7)
    else:
        raise ValueError(f"Unsupported bucket size: {bucket_size}")
    
    return start_time, end_time

def main():
    """Main entry point for metric calculation."""
    args = parse_args()
    logger.info(f"Starting calculation for metric {args.metric_name} ({args.bucket_size})")
    
    try:
        # Load metric configuration
        metric_config = load_metric_config(args.metric_name)
        
        # Calculate time window
        start_time, end_time = calculate_time_window(args.execution_date, args.bucket_size)
        logger.info(f"Calculating for time window: {start_time} to {end_time}")
        
        # Connect to database
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Get calculation SQL
        with open("/opt/metrics/config/calculation_logic.sql", "r") as f:
            sql_template = f.read()
        
        # Format SQL with parameters
        sql = sql_template.format(
            start_time=start_time.isoformat(),
            end_time=end_time.isoformat()
        )
        
        # Execute calculation
        logger.info("Executing calculation query")
        cursor.execute(sql)
        results = cursor.fetchall()
        
        # Process results based on metric type
        if args.metric_type == "counter":
            # Handle counter metric
            value = len(results)  # or some other calculation
        elif args.metric_type == "gauge":
            # Handle gauge metric
            value = results[0][0] if results else 0
        elif args.metric_type == "histogram":
            # Handle histogram metric
            df = pd.DataFrame(results)
            # Perform histogram calculations
            value = json.dumps(df.describe().to_dict())
        else:
            raise ValueError(f"Unsupported metric type: {args.metric_type}")
        
        # Store result in metrics table
        logger.info(f"Storing result: {value}")
        cursor.execute(
            """
            INSERT INTO metrics_data 
            (metric_id, timestamp, bucket_size, bucket_start, bucket_end, value, value_json)
            VALUES 
            ((SELECT id FROM metric_registry WHERE name = %s), 
             %s, %s, %s, %s, %s, %s)
            ON CONFLICT (metric_id, bucket_size, bucket_start, dimensions) 
            DO UPDATE SET value = EXCLUDED.value, value_json = EXCLUDED.value_json
            """,
            (
                args.metric_name,
                datetime.now(),
                args.bucket_size,
                start_time,
                end_time,
                value,
                json.dumps({})  # Placeholder for value_json
            )
        )
        
        # Commit changes
        conn.commit()
        logger.info("Calculation completed successfully")
        
    except Exception as e:
        logger.error(f"Error calculating metric: {e}")
        raise
    finally:
        # Close database connection if it exists
        if 'conn' in locals() and conn is not None:
            conn.close()

if __name__ == "__main__":
    main()
```

## Kubernetes Deployment

The complete Airflow deployment on Kubernetes requires several components:

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: blockchain-metrics
```

### 2. Airflow Components

```yaml
# Airflow Webserver Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airflow-webserver
  namespace: blockchain-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: airflow
      component: webserver
  template:
    metadata:
      labels:
        app: airflow
        component: webserver
    spec:
      containers:
      - name: airflow-webserver
        image: apache/airflow:2.6.1
        command: ["airflow", "webserver"]
        ports:
        - containerPort: 8080
        env:
        - name: AIRFLOW__CORE__EXECUTOR
          value: KubernetesExecutor
        - name: AIRFLOW__KUBERNETES__NAMESPACE
          value: blockchain-metrics
        # Additional environment variables...

# Airflow Scheduler Deployment
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: airflow-scheduler
  namespace: blockchain-metrics
spec:
  serviceName: airflow-scheduler
  replicas: 1
  selector:
    matchLabels:
      app: airflow
      component: scheduler
  template:
    metadata:
      labels:
        app: airflow
        component: scheduler
    spec:
      containers:
      - name: airflow-scheduler
        image: apache/airflow:2.6.1
        command: ["airflow", "scheduler"]
        env:
        - name: AIRFLOW__CORE__EXECUTOR
          value: KubernetesExecutor
        - name: AIRFLOW__KUBERNETES__NAMESPACE
          value: blockchain-metrics
        # Additional environment variables...
```

### 3. Redis Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: blockchain-metrics
spec:
  serviceName: redis
  replicas: 3  # For high availability
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        command: ["redis-server", "/etc/redis/redis.conf"]
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Redis Buffer Durability

The Redis buffer configuration ensures durability with these settings:

```
# redis.conf
appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# RDB snapshots for backup
save 900 1
save 300 10
save 60 10000
```

### Redis Streams Implementation

To ensure data isn't lost during server failures and can be processed reliably:

```python
class RedisBuffer:
    """Redis-based buffer for blockchain events."""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.stream_name = "blockchain_events"
    
    async def add_event(self, event_data):
        """Add an event to the Redis stream."""
        # Convert any non-string values to strings/JSON
        serialized_data = {}
        for key, value in event_data.items():
            if isinstance(value, (dict, list)):
                serialized_data[key] = json.dumps(value)
            else:
                serialized_data[key] = str(value)
        
        # Add to stream with auto-generated ID
        return await self.redis.xadd(
            self.stream_name,
            serialized_data,
            maxlen=100000,  # Limit stream length
            approximate=True
        )
    
    async def create_consumer_group(self, group_name, start_id="$"):
        """Create a consumer group for processing events."""
        try:
            await self.redis.xgroup_create(
                self.stream_name,
                group_name,
                start_id,
                mkstream=True
            )
            return True
        except redis.ResponseError as e:
            if "BUSYGROUP" in str(e):
                # Group already exists
                return False
            raise
    
    async def read_events(self, group_name, consumer_name, count=10, block=2000):
        """Read events from the stream as part of a consumer group."""
        return await self.redis.xreadgroup(
            group_name,
            consumer_name,
            {self.stream_name: ">"},  # > means read new messages
            count=count,
            block=block
        )
    
    async def acknowledge_event(self, group_name, event_id):
        """Acknowledge that an event has been processed."""
        return await self.redis.xack(self.stream_name, group_name, event_id)
    
    async def get_pending_events(self, group_name, count=10):
        """Get list of pending events for a consumer group."""
        return await self.redis.xpending_range(
            self.stream_name,
            group_name,
            min="-",
            max="+",
            count=count
        )
    
    async def claim_pending_events(self, group_name, consumer_name, min_idle_time=30000, count=10):
        """Claim pending events that have been idle too long."""
        # Get list of pending message IDs
        pending = await self.get_pending_events(group_name, count)
        
        if not pending:
            return []
        
        # Extract message IDs that meet the idle time criteria
        message_ids = [
            entry['message_id'] for entry in pending 
            if entry['time_since_delivered'] >= min_idle_time
        ]
        
        if not message_ids:
            return []
        
        # Claim the messages
        return await self.redis.xclaim(
            self.stream_name,
            group_name,
            consumer_name,
            min_idle_time,
            message_ids
        )
    
    async def prune_processed_events(self, last_id):
        """Prune events that have been successfully processed."""
        # Delete all messages up to and including the specified ID
        return await self.redis.xtrim(
            self.stream_name,
            maxlen=0,
            approximate=False,
            minid=last_id
        )
```

## Time Bucket Architecture

The time bucket architecture remains as described previously, with PostgreSQL storing metrics in separate partitions for different time granularities.

## Conclusion

This architecture leverages Airflow on Kubernetes with the KubernetesPodOperator to create a highly scalable, isolated environment for blockchain metric calculations. Each metric runs in its own container, allowing for precise resource allocation and improved reliability.

The Redis buffer ensures data durability during processing with features to prevent data loss during server failures, enable resumption from the last known point, and automatically prune data after successful persistence to PostgreSQL.

By using Kubernetes, we gain several advantages:
- **Scalability**: Automatically scale based on workload
- **Isolation**: Each metric calculation runs in its own environment
- **Resource Efficiency**: Precise allocation of resources per metric
- **Reliability**: Kubernetes handles pod failures and rescheduling
- **Flexibility**: Easy deployment of new metrics with custom images

The Dynamic DAG Builder pattern makes it easy to add new metrics without rebuilding the system, while the time bucket strategy ensures efficient storage and querying across different time scales.
