# airflow_kafka_clickhouse_integration

[link](https://github.com/Lemberg14/kafka_clickhouse_integration/blob/main/Test%20Task%20Appflame%20.pdf) to the test task

Firstly we need to create docker compose file with clickhouse, ubuntu and kafka
```yaml
version: '3.7'

services:
  zookeeper:
    image: zookeeper:latest
    container_name: zookeeper
    restart: always
    networks:
      - kafka_clickhouse_net

  kafka:
    image: bitnami/kafka:latest
    container_name: kafka
    restart: always
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INSIDE://kafka:9093,OUTSIDE://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_LISTENERS: INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper
    networks:
      - kafka_clickhouse_net

  clickhouse:
    image: clickhouse/clickhouse-server
    container_name: clickhouse
    restart: always
    ports:
      - "8123:8123"
      - "9000:9000"
    environment:
      CLICKHOUSE_USER: custom_admin
      CLICKHOUSE_PASSWORD: 'custom_admin'
      CLICKHOUSE_DB: default
      CLICKHOUSE_SERVER_PARAMETERS: --log_level=debug
    networks:
      - kafka_clickhouse_net

  ubuntu:
    image: ubuntu:20.04
    container_name: ubuntu
    command: bash -c "apt-get update && apt-get install -y python3 python3-pip && pip3 install kafka-python && tail -f /dev/null"
    depends_on:
      - kafka
    networks:
      - kafka_clickhouse_net

networks:
  kafka_clickhouse_net:
    driver: bridge
```
Next for airflow docker compose file we need to create custome airflow image
```yaml
FROM apache/airflow:latest
USER airflow
RUN pip install --upgrade pip && \
    pip install clickhouse-connect pip && \
    pip install -U airflow-clickhouse-plugin
USER airflow
```
Based on our custome airflow image we need create second docker compose file for airflow
```yaml
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

# Basic Airflow cluster configuration for CeleryExecutor with Redis and PostgreSQL.
#
# WARNING: This configuration is for local development. Do not use it in a production deployment.
#
# This configuration supports basic configuration using environment variables or an .env file
# The following variables are supported:
#
# AIRFLOW_IMAGE_NAME           - Docker image name used to run Airflow.
#                                Default: apache/airflow:2.10.3
# AIRFLOW_UID                  - User ID in Airflow containers
#                                Default: 50000
# AIRFLOW_PROJ_DIR             - Base path to which all the files will be volumed.
#                                Default: .
# Those configurations are useful mostly in case of standalone testing/running Airflow in test/try-out mode
#
# _AIRFLOW_WWW_USER_USERNAME   - Username for the administrator account (if requested).
#                                Default: airflow
# _AIRFLOW_WWW_USER_PASSWORD   - Password for the administrator account (if requested).
#                                Default: airflow
# _PIP_ADDITIONAL_REQUIREMENTS - Additional PIP requirements to add when starting all containers.
#                                Use this option ONLY for quick checks. Installing requirements at container
#                                startup is done EVERY TIME the service is started.
#                                A better way is to build a custom image or extend the official image
#                                as described in https://airflow.apache.org/docs/docker-stack/build.html.
#                                Default: ''
#
# Feel free to modify this file to suit your needs.
---
x-airflow-common:
  &airflow-common
  # In order to add custom dependencies or upgrade provider packages you can use your extended image.
  # Comment the image line, place your Dockerfile in the directory where you placed the docker-compose.yaml
  # and uncomment the "build" line below, Then run `docker-compose build` to build the images.
  image: ${AIRFLOW_IMAGE_NAME:-airflow1:latest}
  # build: .
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW_CONN_CLICKHOUSE_DEFAULT: http://default:@clickhouse:8123
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'true'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    # yamllint disable rule:line-length
    # Use simple http server on scheduler for health checks
    # See https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html#scheduler-health-check-server
    # yamllint enable rule:line-length
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    # WARNING: Use _PIP_ADDITIONAL_REQUIREMENTS option ONLY for a quick checks
    # for other purpose (development, test and especially production usage) build/extend Airflow image.
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
    # The following line can be used to set a custom config file, stored in the local config folder
    # If you want to use it, outcomment it and replace airflow.cfg with the name of your config file
    # AIRFLOW_CONFIG: '/opt/airflow/config/airflow.cfg'
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    networks:
      - kafka_clickhouse_net
    restart: always

  redis:
    # Redis is limited to 7.2-bookworm due to licencing change
    # https://redis.io/blog/redis-adopts-dual-source-available-licensing/
    image: redis:7.2-bookworm
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50
      start_period: 30s
    networks:
      - kafka_clickhouse_net
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    networks:
      - kafka_clickhouse_net
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    networks:
      - kafka_clickhouse_net
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-worker:
    <<: *airflow-common
    command: celery worker
    healthcheck:
      # yamllint disable rule:line-length
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.providers.celery.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}" || celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    environment:
      <<: *airflow-common-env
      # Required to handle warm shutdown of the celery workers properly
      # See https://airflow.apache.org/docs/docker-stack/entrypoint.html#signal-propagation
      DUMB_INIT_SETSID: "0"
    restart: always
    networks:
      - kafka_clickhouse_net
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    networks:
      - kafka_clickhouse_net
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    # yamllint disable rule:line-length
    command:
      - -c
      - |
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg * 10 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    # yamllint enable rule:line-length
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
      _PIP_ADDITIONAL_REQUIREMENTS: ''
    user: "0:0"
    networks:
      - kafka_clickhouse_net
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}:/sources

  airflow-cli:
    <<: *airflow-common
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
    command:
      - bash
      - -c
      - airflow
    networks:
      - kafka_clickhouse_net

  # You can enable flower by adding "--profile flower" option e.g. docker-compose --profile flower up
  # or by explicitly targeted on the command line e.g. docker-compose up flower.
  # See: https://docs.docker.com/compose/profiles/
  flower:
    <<: *airflow-common
    command: celery flower
    profiles:
      - flower
    ports:
      - "5555:5555"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    networks:
      - kafka_clickhouse_net
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

volumes:
  postgres-db-volume:

networks:
  kafka_clickhouse_net:
    driver: bridge
```
Now we need to run docker

```
docker-compose -f docker-compose1.yml -f docker-compose.yml up
```

Next step we need connect to clickhouse and create out tables

For this we need to list docker containers

```
docker ps -a
```
We have next output
```
CONTAINER ID   IMAGE                          COMMAND                  CREATED        STATUS                      PORTS                                                      NAMES
5aca4c15f8dd   clickhouse/clickhouse-server   "/entrypoint.sh"         19 hours ago   Up 25 minutes               0.0.0.0:8123->8123/tcp, 0.0.0.0:9000->9000/tcp, 9009/tcp   clickhouse
d4ff06b0d89f   airflow1:latest                "/usr/bin/dumb-init …"   19 hours ago   Up 25 minutes (healthy)     8080/tcp                                                   airflow_dockercopy-airflow-triggerer-1
ff5c4f0bdd33   airflow1:latest                "/usr/bin/dumb-init …"   19 hours ago   Up 24 minutes (healthy)     8080/tcp                                                   airflow_dockercopy-airflow-scheduler-1
6669aa035807   airflow1:latest                "/usr/bin/dumb-init …"   19 hours ago   Up 25 minutes (healthy)     0.0.0.0:8080->8080/tcp                                     airflow_dockercopy-airflow-webserver-1
66a40688d358   airflow1:latest                "/usr/bin/dumb-init …"   19 hours ago   Up 25 minutes (healthy)     8080/tcp                                                   airflow_dockercopy-airflow-worker-1
50dbb76bd5d8   ubuntu:20.04                   "bash -c 'apt-get up…"   19 hours ago   Up 25 minutes                                                                          ubuntu
70df954d352e   airflow1:latest                "/bin/bash -c 'if [[…"   19 hours ago   Exited (0) 25 minutes ago                                                              airflow_dockercopy-airflow-init-1
e9dbe868c6d2   bitnami/kafka:latest           "/opt/bitnami/script…"   19 hours ago   Up 25 minutes               0.0.0.0:9092->9092/tcp                                     kafka
0f9beca1dfb7   postgres:13                    "docker-entrypoint.s…"   19 hours ago   Up 25 minutes (healthy)     5432/tcp                                                   airflow_dockercopy-postgres-1
d8dfa2f649f0   redis:7.2-bookworm             "docker-entrypoint.s…"   19 hours ago   Up 25 minutes (healthy)     6379/tcp                                                   airflow_dockercopy-redis-1
8905a7e1ac52   zookeeper:latest               "/docker-entrypoint.…"   19 hours ago   Up 25 minutes               2181/tcp, 2888/tcp, 3888/tcp, 8080/tcp                     zookeeper
```
Next we need to run docker-compose exec -it to connect to clickhouse container

```
docker-compose exec -it 5aca4c15f8dd bash
```

Now we are connected to our clickouse container where we need to launch clickhouse client using next command

```
clickhouse-client
```

Next step we need to create our 4 tables

```sql
CREATE TABLE events (
    ts Int32,
    user_id Int32,    
    operation String,
    status Int32
) Engine = MergeTree()
ORDER BY ts;

CREATE TABLE daily_events_airflow (
    date date,
    operation String,
    count Int32
) Engine = MergeTree()
ORDER BY date;

CREATE TABLE events_queue (
    ts Int32,
    user_id Int32,    
    operation String,
    status Int32
)
ENGINE = Kafka
SETTINGS kafka_broker_list = 'kafka:9093',
       kafka_topic_list = 'events',
       kafka_group_name = 'events_consumer_group1',
       kafka_format = 'JSONEachRow',
       kafka_max_block_size = 1048576;

CREATE MATERIALIZED VIEW events_queue_mv TO events AS
SELECT ts, user_id, operation, status
FROM events_queue;
```

Open new terminal and use docker ps and docker compoase exec to connect to Kafka container

```
docker-compose exec -it e9dbe868c6d2 bash
```

Here we need to create topic

```
kafka-topics.sh --bootstrap-server localhost:9092 --topic events --create --partitions 6 --replication-factor 1
```

We should get next output
```
Created topic events.
```

Open new terminal and use docker ps and docker compoase exec to connect to Ubuntu container

```
docker-compose exec -it 50dbb76bd5d8 bash
```

Here we need to create and run python script which will send every 5 seconds json file to kafka

Create python file
```
nano python.py
```
And insert script it self
```python
from kafka import KafkaProducer
import json
import time

def send_to_kafka(bootstrap_servers, topic, data):
    producer = KafkaProducer(bootstrap_servers=bootstrap_servers,
                             value_serializer=lambda x: json.dumps(x).encode('utf-8'))

    producer.send(topic, value=data)
    producer.flush()
    producer.close()

if __name__ == "__main__":
    bootstrap_servers = 'kafka:9093'  # Kafka service name defined in the Docker Compose file
    topic = 'events'  # Name of the Kafka topic
    data = {
        'ts': 1798128392,
        'user_id': 123,
        'operation': 'like',
        'status': 1
    }  # JSON data to be sent

    while True:
        send_to_kafka(bootstrap_servers, topic, data)
        print("Sent message to Kafka")
        time.sleep(60)  # Send message every 60 seconds
```

To run this script execute 

```
python3 python.py
```
After running the script back to clickhouse terminal and select from events table
```sql
SELECT *
FROM events
```

We should recieve next output

```
SELECT *
FROM events

Query id: e3175f49-5ecc-48ea-b75f-925962ad7e0f

┌─────────ts─┬─user_id─┬─operation─┬─status─┐
│ 1798128392 │     123 │ like      │      1 │
│ 1798128392 │     123 │ like      │      1 │
│ 1798128392 │     123 │ like      │      1 │
└────────────┴─────────┴───────────┴────────┘

3 rows in set. Elapsed: 0.010 sec. 
```
Now we need to create DAG for airflow in folder dags nano test.py
```python
from airflow import DAG
from airflow_clickhouse_plugin.operators.clickhouse import ClickHouseOperator
from airflow.operators.python import PythonOperator
from airflow.utils.dates import days_ago

with DAG(
        dag_id='update_income_aggregate',
        start_date=days_ago(2),
) as dag:
    ClickHouseOperator(
        task_id='update_income_aggregate',
        database='default',
        sql=( 'INSERT INTO daily_events_airflow (date, operation, count) SELECT toDate(FROM_UNIXTIME(ts)) AS date, operation, count() AS count FROM events GROUP BY date, operation'
        ),
        # query_id is templated and allows to quickly identify query in ClickHouse logs
        query_id='{{ ti.dag_id }}-{{ ti.task_id }}-{{ ti.run_id }}-{{ ti.try_number }}',
        clickhouse_conn_id='clickhouse_test',
    ) >> PythonOperator(
        task_id='print_month_income',
        python_callable=lambda task_instance:
            # pulling XCom value and printing it
            print(task_instance.xcom_pull(task_ids='update_income_aggregate')),
    )
```
After creation of dag we need to restart our docker compose 
Next we need to create connection between airflow and clickhouse server
<img width="1799" alt="Screenshot 2024-12-01 at 13 33 33" src="https://github.com/user-attachments/assets/47bfc87a-5f18-4349-b117-94256df6f03d">
Open DAG on webserver 
<img width="1777" alt="Screenshot 2024-12-01 at 13 54 24" src="https://github.com/user-attachments/assets/f44bd0b8-8710-469a-9167-b59f7c04d031">
Run DAG
<img width="1795" alt="Screenshot 2024-12-01 at 13 56 20" src="https://github.com/user-attachments/assets/a2ebf7c8-3884-4344-a06c-dbb73b0ab9bf">
Now go back to clickhouse terminal 
```sql
SELECT *
FROM daily_events_airflow

Query id: 679a0968-3576-4f3b-beb0-180627f4157b

   ┌───────date─┬─operation─┬─count─┐
1. │ 2026-12-12 │ like      │     4 │
   └────────────┴───────────┴───────┘

1 rows in set. Elapsed: 0.014 sec.
```
Now we will create aggregated table using SummingMergeTree
```sql
CREATE TABLE daily_events_ch (
    date Date,
    operation String,
    count UInt32
) ENGINE = SummingMergeTree()
ORDER BY date;

CREATE MATERIALIZED VIEW mv_daily_events_ch TO daily_events_ch AS
SELECT 
    toDate(FROM_UNIXTIME(ts)) AS date,
    operation,
    count() AS count
FROM events
GROUP BY date, operation;
```
Now select from daily_events_ch to see how SummingMergeTree  is implemented
```sql
SELECT *
FROM daily_events_ch

Query id: a8ad79c2-a225-4428-b233-74debb9b7f4f

   ┌───────date─┬─operation─┬─count─┐
1. │ 2026-12-12 │ like      │     4 │
   └────────────┴───────────┴───────┘

1 rows in set. Elapsed: 0.004 sec. 
```
Now going back to ubuntu terminal press CTRL+c to cancel python script to stop it and nano python.py to change ts to 1798128392
After run script again by
```
python3 python.py
```
Now open clickhouse terminal and select from events
```sql
SELECT *
FROM events

Query id: 6cd0360a-2403-4a74-b772-733ec88a6e72

   ┌─────────ts─┬─user_id─┬─operation─┬─status─┐
1. │ 1797052392 │     123 │ like      │      1 │
2. │ 1797052392 │     123 │ like      │      1 │
3. │ 1797052392 │     123 │ like      │      1 │
4. │ 1797052392 │     123 │ like      │      1 │
5. │ 1798128392 │     123 │ like      │      1 │
6. │ 1798128392 │     123 │ like      │      1 │
7. │ 1798128392 │     123 │ like      │      1 │
8. │ 1798128392 │     123 │ like      │      1 │
9. │ 1798128392 │     123 │ like      │      1 │
   └────────────┴─────────┴───────────┴────────┘

9 rows in set. Elapsed: 0.011 sec. 
```
And check how SummingMergeTree works
```sql

SELECT *
FROM daily_events_ch

Query id: fcbc734b-8deb-460e-90c7-8bcb09f13f33

   ┌───────date─┬─operation─┬─count─┐
1. │ 2026-12-12 │ like      │     4 │
2. │ 2026-12-24 │ like      │     3 │
   └────────────┴───────────┴───────┘

2 rows in set. Elapsed: 0.010 sec. 
```
