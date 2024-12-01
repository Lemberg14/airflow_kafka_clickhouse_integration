FROM apache/airflow:latest
USER airflow
RUN pip install --upgrade pip && \
    pip install clickhouse-connect pip && \
    pip install -U airflow-clickhouse-plugin
USER airflow

CREATE TABLE daily_events_airflow (
    date date,
    operation String,
    count Int32
) Engine = MergeTree()
ORDER BY date;