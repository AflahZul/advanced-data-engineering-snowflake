 **DevOps with Snowflake**
 Created by AflahZul, Date: 3rd December

# 🚀 DevOps with Snowflake: DataOps Cheatsheet

## I. DataOps Philosophy and Goals

**DataOps** is a set of philosophies and best practices for delivering and maintaining data pipelines at scale. It applies the principles of DevOps (collaboration, automation, continuous improvement) to the data engineering lifecycle.

| Core Goal | Key Benefit |
| :--- | :--- |
| **Reliability** | Ensure high uptime and data quality through testing and automated rollback capabilities. |
| **Responsiveness** | Rapidly introduce changes and new features (fast time-to-market). |
| **Collaboration** | Enable multiple engineers to work on the pipeline simultaneously using source control. |
| **Modernization** | Integrate advanced tooling (CLI, CI/CD) for resilient, future-proof pipelines. |

## II. Core DataOps Practices

| Practice | Definition | Snowflake Implementation |
| :--- | :--- | :--- |
| **Source Control** | Tracking all changes (SQL, Python, configuration) in a centralized repository (Git). | Achieved by connecting the Snowflake environment to a **GitHub repository** using the native Git Integration. |
| **Database Change Management (DCM)** | Managing the evolution of database objects (tables, views, tasks). | **Declarative Management** is preferred using Snowflake's `CREATE OR ALTER` syntax to safely update objects without destroying their state or history. |
| **CI/CD** | **Continuous Integration** (automated testing) and **Continuous Delivery** (automated deployment). | Orchestrated by **GitHub Actions** which triggers the **Snowflake CLI** to execute version-controlled code. |

## III. Key Tools and Components

| Tool/Component | Role in the Pipeline |
| :--- | :--- |
| **GitHub** | Stores all pipeline code, SQL files, and CI/CD workflow definitions. The single source of truth. |
| **Snowflake CLI** | The command-line executor. Used by CI/CD to connect to Snowflake and run scripts (e.g., `snow git execute`). |
| **GitHub Actions** | The **CI/CD Orchestrator**. Automates testing upon code commit and manages the secure, staged deployment process. |
| **Virtual Warehouse** | The compute resource that executes the pipeline code deployed by the CLI. |

## IV. Critical Workflows and Syntax

### A. Snowflake Git Integration Workflow

The process for connecting the versioned GitHub repository to your Snowflake account:

1.  **Fork** the main repository into your GitHub account.
2.  Generate a **Personal Access Token (PAT)** in GitHub.
3.  Create an **API Integration** in Snowflake to securely store the PAT and define allowed GitHub URLs.
      * *Syntax:* `CREATE API INTEGRATION ... GIT_CREDENTIALS_PAT = '...';`
4.  Create a **Git Repository Object** in Snowflake that references the fork and the API integration.
      * *Syntax:* `CREATE GIT REPOSITORY ... REPOSITORY_URL = '...' ...;`

### B. Database Change Management (DCM) Approaches

| Approach | Focus | Key Snowflake Syntax | Risk/Best Use |
| :--- | :--- | :--- | :--- |
| **Declarative** | Defines the **desired final state** of the object. | `CREATE OR ALTER TASK ... AS ...;` or `CREATE OR REPLACE VIEW ... AS ...;` | **Low Risk.** Preferred for logic/definition changes (Views, Tasks, UDFs) that are easily rebuilt or altered. |
| **Imperative** | Defines the sequential **steps** to reach a state. | `ALTER TABLE ... ADD COLUMN ...;` | **High Risk.** Used only for non-reversible, structural changes to persistent tables (e.g., adding a non-nullable column). |

### C. CI/CD Deployment Workflow

The CD workflow ensures safe deployment from Git $\rightarrow$ Staging $\rightarrow$ Production.

1.  **Code Commit:** Engineer pushes code to a feature branch.
2.  **CI Trigger:** GitHub Actions runs automated tests (syntax, schema checks).
3.  **Staging Deployment (Auto):** On merge to the `dev` or `staging` branch, the CD workflow executes the deployment script against the Staging Snowflake environment using the CLI.
      * *CLI Action Example:* `snow git execute --environment STAGING --file deploy/setup.sql`
4.  **Production Deployment (Gated):** After passing Staging tests, the code is merged to the `main` or `production` branch, triggering deployment to the Production Snowflake environment. This often requires a manual approval gate.

### D. Critical Declarative Syntax

The `CREATE OR ALTER` command is essential for safe, iterative deployment of stateful objects like **Tasks**:

```sql
-- Safe, Idempotent Deployment of a Task
CREATE OR ALTER TASK my_data_pipeline_task
    WAREHOUSE = ETL_WH
    SCHEDULE = 'USING CRON 0 9 * * * America/Los_Angeles' -- New schedule is applied
AS
    CALL my_transformation_sp();

ALTER TASK my_data_pipeline_task RESUME;
```

# 👁️ Observability in Snowflake: Summary Cheatsheet

## I. Observability Pillars and Definition

| Key Concept | Definition | Purpose in Data Engineering |
| :--- | :--- | :--- |
| **Observability** | The ability to understand the internal state of a system (pipeline) by examining its external outputs (telemetry data). | Moves teams from reacting to known failures to proactively diagnosing the *unknown causes* of performance issues and data quality problems. |
| **Logs** | Immutable, time-stamped records of discrete events (the "receipts"). | Tracking execution flow, debugging specific operations, and capturing error messages. |
| **Traces** | A detailed, hierarchical record of a single transaction's journey through multiple steps, linked by Spans and Attributes. | Diagnosing latency, performance bottlenecks, and dependency issues across a distributed pipeline. |
| **Metrics** | Numerical data points reflecting system health (counts, averages, rates). | Dashboards, alerting, and measuring high-level performance indicators (Throughput, Latency). |



## II. Snowflake's Native Observability Framework

Snowflake's framework (built on OpenTelemetry standards) provides frictionless, SQL-driven tools for data teams.

| Snowflake Feature | Telemetry Type | Description |
| :--- | :--- | :--- |
| **Event Tables** | Logs, Traces | Specialized tables that serve as the central repository for custom telemetry data generated by your code. They adhere to **OpenTelemetry (OTEL)** standards. |
| **Log Levels** | Logs | Configurable severity thresholds (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `FATAL`) to manage the volume of records captured in the Event Table. |
| **`SYSTEM$START_SPAN()`** | Traces | Functions used in Stored Procedures (e.g., Python/SQL) to define units of work (Spans) and automatically populate `TRACE_ID` and `SPAN_ID` fields in the Event Table. |
| **Alerts** | Action/Metrics | Serverless objects that execute a conditional SQL query on a schedule. If the condition is met, a defined action is taken. |
| **Notification Integrations** | Action | Secure communication channels (e.g., email, Slack, SNS) configured to send external messages when an Alert is triggered, encouraging immediate team action. |

## III. Key Workflows and Best Practices

### A. Logging Best Practices

1.  **Set Log Level:** Configure the minimum severity level (`ALTER ACCOUNT SET LOG_LEVEL = INFO;`) to avoid excessive DEBUG logs in production.
2.  **Use Native Libraries:** Utilize Python's standard `logging` library within Snowpark Stored Procedures; it automatically writes to the active Event Table.
3.  **Data Protection:** Implement **Row Access Policies** or data masking on the Event Table to protect sensitive data (PII) that might be inadvertently captured in logs.

### B. Alerting Workflow

The system moves from passive monitoring to proactive action:

$$\text{Scheduled Alert Runs} \rightarrow \text{Condition Query Returns Rows} \rightarrow \text{Trigger Action (Stored Procedure)} \rightarrow \text{Log Failure to Audit Table} \rightarrow \text{Send Notification}$$

### C. Integration & Scalability

* **OTEL Standard:** Because Snowflake's telemetry adheres to OpenTelemetry, it ensures easy integration with popular third-party tools (Datadog, Grafana, PagerDuty) for consolidated dashboarding and incident management across the enterprise.
* **Continuous Improvement:** Observability is continuous. Teams must always look for new dimensions to track, new conditions to alert on, and adjust logging verbosity as pipeline requirements evolve.
