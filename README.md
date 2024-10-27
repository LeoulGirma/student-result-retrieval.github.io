# Simulating High-Concurrency Scenarios for Student Result Retrieval 📈

## Table of Contents 📚

1. [Introduction](#1-introduction)
2. [Data Generation and Insertion](#2-data-generation-and-insertion)
3. [API Development and Caching Strategy](#3-api-development-and-caching-strategy)
   - [Building the API](#31-building-the-api)
   - [Loading Data into Redis](#32-loading-data-into-redis)
4. [Performance Optimizations](#4-performance-optimizations)
5. [System Orchestration](#5-system-orchestration)
6. [Infrastructure Components](#6-infrastructure-components)
   - [API Services (api1, api2, api3)](#61-api-services-api1-api2-api3)
   - [NGINX](#62-nginx)
   - [PostgreSQL Database](#63-postgresql-database)
   - [Redis Cache](#64-redis-cache)
   - [Load Generator (k6)](#65-load-generator-k6)
   - [Monitoring Tools](#66-monitoring-tools)
   - [db-seeder](#67-db-seeder)
7. [Load Testing](#7-load-testing)
   - [Objective](#71-objective)
   - [Methodology](#72-methodology)
   - [Results](#73-results)
     - [Key Metrics Overview](#731-key-metrics-overview)
     - [System Resource Utilization](#732-system-resource-utilization)
     - [Insights & Recommendations](#733-insights--recommendations)
8. [Challenges and Optimizations](#8-challenges-and-optimizations)
   - [Initial Issues](#81-initial-issues)
   - [Enhancements Made](#82-enhancements-made)
     - [NGINX Configuration](#821-nginx-configuration)
     - [API Services Optimization](#822-api-services-optimization)
     - [Load Testing Configuration](#823-load-testing-configuration)
     - [Monitoring and Profiling](#824-monitoring-and-profiling)
   - [Implementing Caching Mechanisms](#83-implementing-caching-mechanisms)
   - [Horizontal Scaling](#84-horizontal-scaling)
9. [Conclusion](#9-conclusion)

---

## 1. Introduction 📝

When students eagerly check their exam results, slow response times can lead to frustration and anxiety. Inspired by Chapimagna's post highlighting these issues, I set out to simulate real-world scenarios to identify potential bottlenecks and develop a scalable solution. My objective was to create a system capable of handling high traffic volumes without compromising performance, ensuring that students can access their results quickly and reliably.

To achieve this, I focused on replicating the conditions of a high-concurrency environment where hundreds of thousands of students might be accessing their results simultaneously. This involved generating a substantial dataset, developing efficient APIs, implementing robust caching mechanisms, and optimizing each component of the system for maximum performance. By thoroughly testing and refining the infrastructure, I aimed to provide insights into how such systems can be built and scaled effectively.

## 2. Data Generation and Insertion 🗄️

To mimic a realistic environment, I estimated that approximately **674,000** students took the exam. I created tables in a **PostgreSQL** database to store student information and their results for both natural and social science streams. Generating and inserting such a large volume of data is significant, so I employed **batch inserts** to reduce the number of database transactions. By parallelizing data insertions, I optimized the data seeding process and minimized the load on the database server.

## 3. API Development and Caching Strategy 🚀

### 3.1. Building the API 🛠️

The next step was to develop an **API** that responds with student results. Implemented in **Go**, the API services are stateless to facilitate horizontal scaling and are designed to handle multiple concurrent requests efficiently using **goroutines**.

Since most student data doesn't change frequently, I decided to cache the responses directly in a **Redis** database using the student ID as the key. This approach reduces the need to query the database repeatedly, thereby enhancing performance and reducing latency.

### 3.2. Loading Data into Redis ⚡

To efficiently load data into Redis, I:

- **Used pipelining** to batch Redis commands, minimizing round-trip time.
- **Parallelized cache loading** using goroutines in Go to speed up the process.
- **Implemented mechanisms** to check for existing data to prevent unnecessary writes.
- **Synchronized processes** using primitives like `sync.WaitGroup` to coordinate data seeding and caching.

These strategies ensured that the cache was preloaded only after the database seeding was complete, maintaining data consistency between PostgreSQL and Redis. If the data was already cached, the API would skip the caching process upon startup.

## 4. Performance Optimizations ⚙️

To enhance overall performance, I:

- **Optimized data serialization/deserialization** by choosing efficient encoding methods.
- Utilized **appropriate data structures** for efficient caching, such as Redis hashes.
- **Implemented connection pooling** using `pgxpool` in Go for efficient database connections.
- **Used Go's `pprof` tool** to identify and optimize CPU-intensive operations within the API services.

These optimizations reduced overhead, improved response times, and ensured efficient resource utilization across the system.

## 5. System Orchestration 🧩

I orchestrated the entire system using **Docker Compose**, which included:

- **Three API services** (`api1`, `api2`, `api3`) implemented in Go, with **NGINX** serving as a load balancer.
- **PostgreSQL** and **Redis** services for data storage and caching.
- **Grafana** and **Prometheus** with Node Exporter and cAdvisor for system monitoring.
- **db-seeder** service to initialize the PostgreSQL database and preload Redis with necessary data.
- **Load Generator (k6)** to simulate user load during testing.

This setup allowed for easy deployment, scaling, and management of all components, providing a consistent environment for testing and development.

## 6. Infrastructure Components 🏗️

### 6.1. API Services (api1, api2, api3) 🖥️

- **Stateless Go applications** designed for horizontal scaling.
- Interact with **PostgreSQL** for persistent storage and **Redis** for caching.
- Handle multiple concurrent requests efficiently using **goroutines**.
- Expose custom metrics for monitoring performance and resource utilization.

### 6.2. NGINX 🌐

- Acts as a **reverse proxy and load balancer**.
- Distributes incoming traffic across API service instances.
- Configured with optimized settings to handle high concurrency efficiently.
- **Handles SSL termination** (if needed) and serves as the entry point for load tests.

### 6.3. PostgreSQL Database 🗃️

- Serves as the **primary data store**.
- Populated with initial data via the **db-seeder**.
- Configured for efficient connection handling and query performance.

### 6.4. Redis Cache 🧠

- Provides **in-memory caching** to enhance response times.
- Preloaded with frequently accessed data to reduce database load.
- Managed by the **db-seeder** for initial data population.

### 6.5. Load Generator (k6) 🏃

- Executes the `real_k6.js` script to simulate user load.
- Configured to generate a high volume of requests for stress-testing.
- Runs outside of Docker to maximize available resources and reduce overhead.

### 6.6. Monitoring Tools 📊

- **Prometheus** and **Grafana**: Collect and visualize metrics from all services.
- **cAdvisor**: Monitors Docker container resource usage.
- **NGINX Prometheus Exporter**: Exposes NGINX metrics for detailed analysis.
- **Node Exporter**: Provides hardware and OS metrics exposed by *NIX kernels.

### 6.7. db-seeder 🌱

- A Go-based service responsible for **seeding the PostgreSQL database**.
- **Preloads Redis** with necessary data.
- Ensures data consistency and readiness before load testing begins.
- Uses synchronization primitives to coordinate seeding and caching processes.

## 7. Load Testing 🧪

### 7.1. Objective 🎯

The load test aimed to simulate realistic and high-concurrency user interactions with the API services, ensuring that the system could handle substantial traffic without compromising response times or reliability.

### 7.2. Methodology 📋

- **Endpoint Tested**: `http://localhost:8081/student/{studentId}`, where `{studentId}` is a randomly generated integer between **10,000** and **674,823**.
- **Load Simulation**: Ramped up the number of **virtual users (VUs)**, peaking at **3,000 concurrent users**, and gradually released the load afterward.
- **Execution**: Used **k6** for load testing, running the tests outside of Docker to reduce overhead.
- **Duration**: The test simulated a sustained high-load scenario over a significant period.

### 7.3. Results 📈

#### 7.3.1. Key Metrics Overview 📊

- **Success Rate**: 100% (**1,382,231** successful requests, **0** failures)
- **Request Duration**:
  - **Average**: **47.88 ms**
  - **Median**: **19.47 ms**
  - **90th Percentile**: **108 ms**
  - **95th Percentile**: **190 ms**
  - **Max**: **2.13 s** (few outliers)
- **Data Throughput**:
  - **Data Received**: **750 MB** (~781 kB/s)
  - **Data Sent**: **130 MB** (~135 kB/s)
- **Request Blocking**: Very low (avg **19.79 µs**), minimal connection delay (**5.8 µs**)
- **Virtual Users (VUs)**: Handled **3,000 VUs** smoothly, sustaining ~**1,438 requests/s**

#### 7.3.2. System Resource Utilization 🖥️

- **Testing Machine**: Core i5 10300H (4 cores, 8 threads @ 2.4 GHz) with **16 GB DDR4 RAM**
- **CPU Usage**: Peaked at **500%**, indicating utilization across 5 logical processors (since 100% represents one logical processor). Stayed mostly between **100%** to **300%**.
- **Memory Usage**: Stable around **870 MB**, well within the allocated **7.47 GB**.
- **CPU Graph**: Showed spikes corresponding to load but remained within acceptable limits.

#### 7.3.3. Insights & Recommendations 💡

- **Consistent Performance**: The system showed low latency and minimal variance, indicating stability under high load.
- **Latency Management**: Monitor the maximum request duration (**2.13 s**) for potential bottlenecks and consider further optimizations if latency spikes increase.
- **Scalability Testing**: Testing with higher loads for longer periods would be beneficial. Performing these tests on a more powerful machine would provide more accurate insights into how the system behaves under extreme conditions.
- **Resource Utilization**: Efficient CPU and memory usage suggest that the system scales well with available resources.

## 8. Challenges and Optimizations 🛠️

### 8.1. Initial Issues ⚠️

- **Inefficient NGINX Configuration**: The initial `nginx.conf` was minimal and not optimized for high concurrency, leading to inefficient connection handling and increased CPU usage.
- **Resource Limitations**: Faced issues creating **3,000 virtual users** due to RAM limitations on the testing machine.
- **High CPU Usage**: Observed peaks above **500%** CPU usage, indicating the need for optimization.

### 8.2. Enhancements Made 🏋️

#### 8.2.1. NGINX Configuration 🛡️

- **Worker Processes and Connections**:
  - Set `worker_processes auto;` to match the number of CPU cores.
  - Increased `worker_connections` to **4096** to handle more simultaneous connections.
- **Event Handling**:
  - Used `epoll` for efficient event handling on Linux systems.
- **Gzip Compression**:
  - Enabled Gzip and set `gzip_comp_level 1;` to reduce CPU overhead.
- **Proxy Settings**:
  - Adjusted timeouts and buffering to handle slow responses and prevent buffer overflows.
  - Configured upstream servers with `max_fails` and `fail_timeout` to handle failed instances gracefully.
- **Result**:
  - Improved NGINX's ability to handle high concurrency efficiently.
  - Reduced CPU usage attributed to NGINX.
  - Eliminated failed requests due to NGINX misconfigurations.

#### 8.2.2. API Services Optimization 🚀

- **Connection Pooling**:
  - Implemented using `pgxpool` in Go for efficient database connections.
- **Code Profiling and Optimization**:
  - Used Go's `pprof` tool to identify and optimize CPU-intensive operations.
  - Optimized algorithms and data handling to reduce unnecessary computations.
- **Caching Implementation**:
  - Introduced in-memory caching with Redis within API services.
  - Cached database query results and computation-heavy data.
- **Concurrency Handling**:
  - Ensured efficient handling of concurrent requests using goroutines.
- **Result**:
  - Reduced CPU usage in API services.
  - Improved response times and throughput.
  - Achieved 100% success rate in subsequent load tests.

#### 8.2.3. Load Testing Configuration ⚙️

- **Enhanced k6 Script**:
  - Included custom metrics and thresholds for deeper insights.
  - Implemented detailed logging for failed requests (though failures were eliminated later).
- **Adjusted VUs**:
  - Scaled virtual users gradually to monitor system behavior and resource utilization.
  - Avoided overloading the system abruptly.
- **Ran Load Tests Outside Docker**:
  - Executed k6 on the host machine to reduce Docker overhead.
- **Result**:
  - Provided more accurate testing results.
  - Maximized resource availability for the load generator.

#### 8.2.4. Monitoring and Profiling 🔍

- **Prometheus and Grafana**:
  - Collected metrics from NGINX, API services, PostgreSQL, and Docker containers.
  - Visualized metrics through Grafana dashboards for real-time monitoring.
- **API Service Metrics**:
  - Instrumented API services to expose custom metrics (e.g., request counts, error rates).
- **Result**:
  - Enabled proactive identification of performance issues.
  - Facilitated data-driven optimization decisions.
  - Improved overall system observability.

### 8.3. Implementing Caching Mechanisms 🧩

- **Server-Side Caching with NGINX**:
  - Configured NGINX to cache responses for frequently accessed endpoints.
  - Reduced the load on API services by serving cached content directly.
- **Application-Level Caching**:
  - Used Redis as an in-memory cache within API services.
  - Cached database query results and computation-heavy data.
- **Result**:
  - Decreased response times.
  - Reduced CPU and database load.
  - Improved system scalability.

### 8.4. Horizontal Scaling 📈

- **Increased API Service Replicas**:
  - Scaled up the number of API service instances using Docker Compose scaling.
- **Load Balancing**:
  - Ensured NGINX efficiently distributed traffic across all instances.
- **Future Scope**:
  - Considering transitioning to **Kubernetes** for advanced orchestration and scalability.
- **Result**:
  - Distributed load more evenly.
  - Reduced CPU usage per container.
  - Enhanced system capacity to handle higher concurrency.

## 9. Conclusion 🏁

I am satisfied with the outcomes of this project. By simulating high-concurrency scenarios and optimizing each component of the system, I was able to create a scalable and efficient environment capable of handling substantial traffic. The successful handling of high concurrency with optimal performance metrics suggests that the application is ready for high-traffic production scenarios.

Testing with higher loads for longer periods would be beneficial to further validate the system's robustness and explore its upper capacity limits. Performing these tests on a more powerful machine would provide more accurate insights into how the system behaves under extreme conditions, ensuring it can maintain performance and reliability at scale.

---

**Note**: Optimizing individual components like **NGINX**, **API services**, the **database**, and **caching mechanisms** significantly contributed to the overall system performance. Continuous monitoring and iterative enhancements were key to achieving the desired scalability and reliability.

---
