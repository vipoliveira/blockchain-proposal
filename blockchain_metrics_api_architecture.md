# Blockchain Metrics API Architecture

## Overview

This document outlines the architecture for the API layer of our blockchain metrics framework. The API serves as the interface between the metrics data stored in PostgreSQL and external consumers such as dashboards, analytics tools, and third-party integrations. Built with Python and deployed on Kubernetes, this API layer provides efficient, secure access to our time-bucketed metrics data.

## Architecture Design

### API Technology Selection: FastAPI + GraphQL

For our blockchain metrics API, we propose using **FastAPI** as the underlying web framework with **GraphQL** as the query language, specifically using the **Strawberry GraphQL** library for Python.

#### Why FastAPI?

1. **Performance**: FastAPI is one of the fastest Python web frameworks, built on Starlette and Pydantic
2. **Type Safety**: Built-in data validation and type annotations reduce errors
3. **Async Support**: Native asynchronous request handling for optimal database I/O operations
4. **Documentation**: Automatic OpenAPI and Swagger UI generation
5. **Kubernetes-Friendly**: Lightweight, containerizable, and works well with k8s health probes

#### Why GraphQL?

For a metrics API specifically, GraphQL offers significant advantages over REST:

1. **Flexible Data Fetching**: Clients can request exactly the metrics and dimensions they need
2. **Reduced Network Overhead**: No over-fetching of data, particularly important for dashboard applications
3. **Strong Typing**: Schema-based typing aligns well with our metrics registry
4. **Efficient Aggregations**: Complex metric combinations can be retrieved in a single request
5. **Evolving Requirements**: New metrics can be added without versioning the API
6. **Introspection**: Clients can discover available metrics and their structures

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                    │
│                                                         │
│  ┌─────────────┐        ┌─────────────────────────────┐ │
│  │ Ingress     │        │       API Deployment        │ │
│  │ Controller  │        │                             │ │
│  │             │        │  ┌─────────┐  ┌─────────┐   │ │
│  └──────┬──────┘        │  │FastAPI  │  │GraphQL  │   │ │
│         │               │  │Pod      │  │Pod      │   │ │
│         │               │  └─────────┘  └─────────┘   │ │
│         │               │                             │ │
│         └───────────────┼─▶ ┌─────────┐  ┌─────────┐  │ │
│                         │  │FastAPI  │  │GraphQL  │  │ │
│                         │  │Pod      │  │Pod      │  │ │
│                         │  └─────────┘  └─────────┘  │ │
│                         │                             │ │
│                         └─────────────┬───────────────┘ │
│                                       │                 │
│  ┌─────────────┐                      │                 │
│  │             │                      │                 │
│  │ Redis Cache ◀──────────────────────┘                 │
│  │             │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │             │                                        │
│  │ PostgreSQL  │                                        │
│  │ Metrics DB  │                                        │
│  │             │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

### Key Components:

1. **API Pods**:
   - Horizontally scalable FastAPI instances
   - Deployed with a Kubernetes Deployment resource
   - Autoscaling based on CPU/memory metrics
   - Resource limits tuned for GraphQL query handling

2. **Ingress Controller**:
   - NGINX Ingress for TLS termination and routing
   - Rate limiting to prevent API abuse
   - Path-based routing for API versions and documentation

3. **Redis Cache**:
   - In-memory caching of frequent GraphQL queries
   - Automatic invalidation when metrics are updated
   - Configurable TTL based on metric update frequency

4. **PostgreSQL Connection Pool**:
   - Connection pooling to the metrics database
   - Read replicas for high-volume queries
   - Async drivers for efficient database access

## API Design

### GraphQL Schema Design

Our GraphQL schema will be built around the core concept of metrics and time buckets:

```graphql
type Metric {
  id: ID!
  name: String!
  description: String
  metricType: MetricType!
  value: Float
  dimensions: [Dimension!]
  timestamp: DateTime!
  bucketSize: BucketSize!
  bucketStart: DateTime!
  bucketEnd: DateTime!
}

enum MetricType {
  COUNTER
  GAUGE
  HISTOGRAM
}

enum BucketSize {
  HOURLY
  DAILY
  WEEKLY
}

type Dimension {
  name: String!
  value: String!
}

type Query {
  # Get a single metric by name and time range
  metric(
    name: String!, 
    bucketSize: BucketSize!, 
    from: DateTime!, 
    to: DateTime!,
    dimensions: [DimensionInput!]
  ): [Metric!]!
  
  # Get multiple metrics at once (for dashboards)
  metrics(
    names: [String!]!, 
    bucketSize: BucketSize!, 
    from: DateTime!, 
    to: DateTime!,
    dimensions: [DimensionInput!]
  ): [Metric!]!
  
  # Get available metrics for discovery
  availableMetrics: [MetricDefinition!]!
}

type MetricDefinition {
  name: String!
  description: String
  metricType: MetricType!
  availableDimensions: [String!]!
  availableBuckets: [BucketSize!]!
}

input DimensionInput {
  name: String!
  value: String!
}
```

### Implementation with FastAPI and Strawberry GraphQL

```python
import strawberry
from strawberry.fastapi import GraphQLRouter
from fastapi import FastAPI, Depends
from typing import List, Optional
import asyncpg
from datetime import datetime
import redis

# Setup FastAPI application
app = FastAPI(title="Blockchain Metrics API")

# Database connection
async def get_db_pool():
    return await asyncpg.create_pool(
        host="postgres-metrics",
        database="metrics",
        user="metrics_user",
        password="password"  # Use k8s secrets in production
    )

# Redis cache
redis_client = redis.Redis(host="redis-cache", port=6379, db=0)

# GraphQL types
@strawberry.enum
class MetricType:
    COUNTER = "counter"
    GAUGE = "gauge"
    HISTOGRAM = "histogram"

@strawberry.enum
class BucketSize:
    HOURLY = "hourly"
    DAILY = "daily"
    WEEKLY = "weekly"

@strawberry.type
class Dimension:
    name: str
    value: str

@strawberry.input
class DimensionInput:
    name: str
    value: str

@strawberry.type
class Metric:
    id: strawberry.ID
    name: str
    description: Optional[str]
    metric_type: MetricType
    value: float
    dimensions: List[Dimension]
    timestamp: datetime
    bucket_size: BucketSize
    bucket_start: datetime
    bucket_end: datetime

@strawberry.type
class MetricDefinition:
    name: str
    description: Optional[str]
    metric_type: MetricType
    available_dimensions: List[str]
    available_buckets: List[BucketSize]

# Query resolvers
async def get_metric(
    name: str,
    bucket_size: BucketSize,
    from_date: datetime,
    to_date: datetime,
    dimensions: Optional[List[DimensionInput]] = None
) -> List[Metric]:
    # Generate cache key
    cache_key = f"metric:{name}:{bucket_size}:{from_date}:{to_date}:{dimensions}"
    
    # Check cache first
    cached = redis_client.get(cache_key)
    if cached:
        return cached
    
    # Query database if not in cache
    pool = await get_db_pool()
    async with pool.acquire() as conn:
        # Build query based on dimensions
        if dimensions:
            dimension_conditions = []
            for dim in dimensions:
                dimension_conditions.append(
                    f"dimensions @> '{{\"{dim.name}\": \"{dim.value}\"}}'"
                )
            dimension_query = " AND " + " AND ".join(dimension_conditions)
        else:
            dimension_query = ""
        
        query = f"""
        SELECT 
            m.id, m.name, mr.description, mr.metric_type, m.value, 
            m.dimensions, m.timestamp, m.bucket_size, m.bucket_start, m.bucket_end
        FROM 
            metrics_data m
        JOIN 
            metric_registry mr ON m.metric_id = mr.id
        WHERE 
            mr.name = $1 
            AND m.bucket_size = $2
            AND m.bucket_start >= $3
            AND m.bucket_end <= $4
            {dimension_query}
        ORDER BY 
            m.bucket_start
        """
        
        rows = await conn.fetch(query, name, bucket_size, from_date, to_date)
        
        # Convert to Metric objects
        metrics = []
        for row in rows:
            metrics.append(
                Metric(
                    id=row['id'],
                    name=row['name'],
                    description=row['description'],
                    metric_type=MetricType(row['metric_type']),
                    value=row['value'],
                    dimensions=[
                        Dimension(name=k, value=v) 
                        for k, v in row['dimensions'].items()
                    ],
                    timestamp=row['timestamp'],
                    bucket_size=BucketSize(row['bucket_size']),
                    bucket_start=row['bucket_start'],
                    bucket_end=row['bucket_end']
                )
            )
        
        # Cache result
        redis_client.setex(
            cache_key, 
            # TTL based on bucket size
            60 * 5 if bucket_size == BucketSize.HOURLY else 60 * 30, 
            metrics
        )
        
        return metrics

# GraphQL Schema
@strawberry.type
class Query:
    metric: List[Metric] = strawberry.field(resolver=get_metric)
    # Add other resolvers for metrics and availableMetrics

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema)

# Add GraphQL endpoint to FastAPI
app.include_router(graphql_app, prefix="/graphql")

# Add health check for Kubernetes
@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

## Authentication and Authorization

### Authentication Mechanisms

The API will support multiple authentication methods:

1. **API Keys**: For dashboard and internal service integrations
   - Stored securely in Kubernetes secrets
   - Rate-limited per key
   - Scoped to specific metrics

2. **OAuth2/OIDC**: For user-based access
   - Integration with existing identity providers
   - JWT validation
   - User-specific permissions

3. **Service-to-Service**: For internal microservices
   - Kubernetes service accounts
   - mTLS for secure communication

### Authorization Model

Authorization will be based on:

1. **Metric-level permissions**: Control access to specific metrics
2. **Dimension-level permissions**: Restrict access to certain dimensions
3. **Time-range restrictions**: Limit historical data access
4. **Rate limiting**: Prevent abuse based on client identity

Implementation using a Policy Enforcement Point pattern:

```python
from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def get_current_client(api_key: str = Security(api_key_header)):
    # Validate API key against database or k8s secret
    if not is_valid_api_key(api_key):
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    # Get client permissions
    client = get_client_by_api_key(api_key)
    return client

@app.middleware("http")
async def authorization_middleware(request, call_next):
    # Extract client from request context
    client = request.state.client
    
    # For GraphQL requests, parse the query and check permissions
    if request.url.path == "/graphql" and request.method == "POST":
        body = await request.json()
        query = body.get("query", "")
        
        # Parse GraphQL query to extract requested metrics
        requested_metrics = parse_graphql_query(query)
        
        # Check if client has access to all requested metrics
        if not all(has_access(client, metric) for metric in requested_metrics):
            return JSONResponse(
                status_code=403,
                content={"detail": "Not authorized to access some metrics"}
            )
    
    response = await call_next(request)
    return response
```

## Caching Strategy

For optimal performance, we implement a multi-level caching strategy:

1. **Query-Level Caching**:
   - Cache complete GraphQL query results based on query+variables hash
   - Different TTL based on bucket size (shorter for hourly, longer for weekly)
   - Automatic invalidation when new metrics are calculated

2. **Dataloader Pattern**:
   - Batch and cache database requests within a single request lifecycle
   - Prevents N+1 query problems common in GraphQL

3. **Redis as Distributed Cache**:
   - Shared cache across API instances
   - Cache fragments for common metric combinations
   - Cache dashboard-specific aggregated results

4. **Client-Side Caching**:
   - HTTP cache headers for static resources
   - GraphQL automatic persisted queries for repeat dashboard views

Implementation with Redis:

```python
import hashlib
import json
from functools import wraps

def cached_resolver(ttl_seconds=300):
    """Decorator for caching GraphQL resolvers."""
    def decorator(resolver_func):
        @wraps(resolver_func)
        async def wrapper(*args, **kwargs):
            # Create a cache key from function name and arguments
            key_parts = [resolver_func.__name__]
            key_parts.extend([str(arg) for arg in args])
            key_parts.extend([f"{k}:{v}" for k, v in sorted(kwargs.items())])
            
            # Create hash of the key parts
            key = hashlib.md5(":".join(key_parts).encode()).hexdigest()
            
            # Check if result is in cache
            cached = redis_client.get(f"resolver:{key}")
            if cached:
                return json.loads(cached)
            
            # Execute resolver and cache result
            result = await resolver_func(*args, **kwargs)
            redis_client.setex(f"resolver:{key}", ttl_seconds, json.dumps(result))
            
            return result
        return wrapper
    return decorator

# Usage
@strawberry.field
@cached_resolver(ttl_seconds=300)
async def get_metrics(
    names: List[str],
    bucket_size: BucketSize,
    from_date: datetime,
    to_date: datetime,
    dimensions: Optional[List[DimensionInput]] = None
) -> List[Metric]:
    # Implementation
    pass
```

## Performance Optimizations

1. **Database Query Optimization**:
   - Denormalized tables for fast access patterns
   - Materialized views for common aggregations
   - Partitioning by time bucket
   - Indexes on common query dimensions

2. **GraphQL-Specific Optimizations**:
   - Query complexity analysis to prevent expensive queries
   - Depth limiting to prevent deeply nested queries
   - Automatic persisted queries to reduce payload size

3. **Asynchronous Processing**:
   - Async database access with connection pooling
   - Non-blocking I/O throughout the API stack

4. **Kubernetes Resources**:
   - Horizontal Pod Autoscaling based on request volume
   - Vertical Pod Autoscaling for resource optimization
   - Pod Disruption Budgets for stability

## Kubernetes Deployment

### Deployment Manifest Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-api
  namespace: blockchain-metrics
spec:
  replicas: 3
  selector:
    matchLabels:
      app: metrics-api
  template:
    metadata:
      labels:
        app: metrics-api
    spec:
      containers:
      - name: api
        image: metrics-api:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        env:
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              name: metrics-config
              key: postgres_host
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: metrics-config
              key: redis_host
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-api
  namespace: blockchain-metrics
spec:
  selector:
    app: metrics-api
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: metrics-api-ingress
  namespace: blockchain-metrics
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit-rps: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: api.metrics.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: metrics-api
            port:
              number: 80
  tls:
  - hosts:
    - api.metrics.example.com
    secretName: metrics-api-tls
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: metrics-api-hpa
  namespace: blockchain-metrics
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: metrics-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## API Usage Examples

### Querying a Single Metric

```graphql
query GetTransactionCount {
  metric(
    name: "transaction_count",
    bucketSize: HOURLY,
    from: "2025-10-01T00:00:00Z",
    to: "2025-10-02T00:00:00Z",
    dimensions: [
      { name: "blockchain", value: "ethereum" }
    ]
  ) {
    name
    value
    bucketStart
    bucketEnd
    dimensions {
      name
      value
    }
  }
}
```

### Dashboard Multi-Metric Query

```graphql
query DashboardMetrics {
  metrics(
    names: ["transaction_count", "gas_usage", "active_addresses"],
    bucketSize: DAILY,
    from: "2025-09-01T00:00:00Z",
    to: "2025-10-01T00:00:00Z",
    dimensions: [
      { name: "blockchain", value: "ethereum" }
    ]
  ) {
    name
    value
    bucketStart
    metricType
    dimensions {
      name
      value
    }
  }
}
```

### Discovering Available Metrics

```graphql
query DiscoverMetrics {
  availableMetrics {
    name
    description
    metricType
    availableDimensions
    availableBuckets
  }
}
```

## Future Enhancements

1. **Subscriptions**: Add GraphQL subscriptions for real-time metric updates
2. **Metric Aggregations**: Support for on-the-fly aggregations across dimensions
3. **Custom Metrics**: Allow clients to define and save custom metric calculations
4. **Data Export**: Implement bulk export functionality for analytics tools
5. **Multi-Region Deployment**: Geo-distributed API endpoints for global access

## Conclusion

This GraphQL-based API architecture for our blockchain metrics provides an efficient, flexible interface for various consumers. By leveraging FastAPI with GraphQL, deployed on Kubernetes, we achieve a scalable solution that can grow with our metrics framework. The multi-tiered caching strategy ensures optimal performance, while the comprehensive authentication and authorization mechanisms provide appropriate security controls.

The architecture aligns with the Lambda architecture of our metrics framework, allowing efficient access to time-bucketed data across various dimensions. The GraphQL interface is particularly well-suited for dashboard applications, enabling them to fetch precisely the data they need in a single request, reducing network overhead and improving user experience.
