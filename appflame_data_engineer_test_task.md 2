# Appflame Test Task

## Objective
Deploy and integrate a system involving Kafka (or an alternative message broker), ClickHouse, and Apache Airflow. Demonstrate skills in containerization, data pipeline setup, and aggregation using Airflow and ClickHouse.

---

## Task Description
Using **Docker Compose**, set up a system including the following components:
1. **ClickHouse**: A database.
2. **Kafka** (or an alternative message broker you have experience with): A message broker with an optional UI
3. **Apache Airflow**: For orchestrating workflows.

---

## Steps

1. **Message Broker Setup**:
   - If using Kafka:
     - Create a Kafka topic named `events`.
     - Write JSON messages to the topic with the following structure:
       ```json
       {
           "ts": 1798128392,
           "user_id": 123,
           "operation": "like",
           "status": 1
       }
       ```
   - If using another message broker:
     - Configure a comparable setup (e.g., topic/queue) for handling the same JSON structure.
     - Document how the system can integrate this broker with ClickHouse.

2. **ClickHouse Setup**:
   - Create a **MergeTree** table named `events` with the following columns:
     - `ts`: Timestamp.
     - `user_id`: Integer.
     - `operation`: String.
     - `status`: Integer.
   - Configure the chosen message broker (Kafka or alternative) and Materialized View to automatically sync data into the `events` table.

3. **Aggregation Using Airflow**:
   - Create an Airflow DAG that:
     - Aggregates data from the `events` table using the following fields:
       - `date` (calculated from `ts`).
       - `operation`.
       - `count` (number of events).
     - Inserts the results into a new table named `daily_events_airflow` with these columns:
       - `date`: Date.
       - `operation`: String.
       - `count`: Integer.

4. **Aggregation Using ClickHouse**:
   - Implement the same aggregated table (`daily_events_ch`), but using:
     - **SummingMergeTree** or **AggregatingMergeTree**.
     - A Materialized View to automatically populate the table based on data in `events`.

   The table should have the following columns:
   - `date`: Date.
   - `operation`: String.
   - `count`: Integer.

5. **Data Loading**:
   - Simulate several days of data by writing messages to the message broker (Kafka or alternative) to populate both the `events` and aggregated tables.

---

## Bonus Task
1. **Replicated ClickHouse**:
   - Set up ClickHouse with 2-3 replicas.
   - Use **ReplicatedMergeTree**, **ReplicatedSummingMergeTree**, or **ReplicatedAggregatingMergeTree** in place of the single-node counterparts.

2. **Web Application**:
   - The application should display a bar chart showing the count of events per day for each operation.
   - Use both aggregated tables (`daily_events_airflow` and `daily_events_ch`) as data sources for comparison.
   - Add filters to the web application to:
     - Filter events by `operation`.
     - Filter events by a date range.

---

## Expected Deliverables
1. **Docker Compose file** to spin up the entire system.
2. Configuration files/scripts for:
   - Message broker topic/queue creation and data loading.
   - ClickHouse table and Materialized View setup.
3. **Airflow DAG file** that implements aggregation via SQL query.
4. Clear instructions for running and verifying the solution.
5. (Bonus) Web application code and replicated ClickHouse setup if implemented.

