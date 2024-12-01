FROM apache/airflow:latest
USER airflow
RUN pip install --upgrade pip && \
    pip install clickhouse-connect pip && \
    pip install -U airflow-clickhouse-plugin
USER airflow

