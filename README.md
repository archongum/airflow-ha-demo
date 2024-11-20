# Airflow HA Demo

2 webserver (PG) + 2 scheduler (PG) + 2 worker ((LocalExecutor), (PG, CeleryExecutor))

## Preparation

### Database: PG

```sh
# Install PG
mkdir -p ./pg/pgdata && chown 999 ./pg/pgdata
docker run \
-d \
--name pg \
--hostname pg \
--restart=always \
-p 5432:5432 \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-e PGPORT=5432 \
-e PGDATA=/data \
-v $(pwd)/pg/pgdata:/data \
postgres:16

# Setup database and user for airflow
docker exec -it -u postgres pg psql -c "CREATE USER airflow PASSWORD 'airflow123'"
docker exec -it -u postgres pg psql -c "CREATE DATABASE airflow OWNER airflow"
```

### (Optional) Logging Storage: S3

```sh
# Install MinIO
mkdir -p ./s3/
docker run \
-d \
--name minio \
--hostname minio \
--restart=always \
-p 9000:9000 \
-p 9001:9001 \
-e MINIO_ROOT_USER=minioadmin \
-e MINIO_ROOT_PASSWORD=minioadmin \
-v $(pwd)/s3/data:/data \
minio/minio server /data --console-address ":9001"

# Setup bucket for airflow logging
docker exec -it minio mc alias set local http://localhost:9000 minioadmin minioadmin
docker exec -it minio mc mb local/airflow-log
```

### (Optional) MQ: RabbitMQ

> Only for CeleryExecutor

```sh
mkdir -p ./mq
docker run \
-d \
--name mq \
--hostname mq \
--restart=always \
-p 5672:5672 \
-p 15672:15672 \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=admin123 \
-e RABBITMQ_DEFAULT_VHOST=airflow_celery \
-v $(pwd)/mq/data:/var/lib/rabbitmq/mnesia \
rabbitmq:3-management
```

## Airflow: node-01

### Installation

```sh
# python venv
python -m venv ./node-01/airflow
source ./node-01/airflow/bin/activate

# install airflow
AIRFLOW_VERSION=2.10.3
PYTHON_VERSION="$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
pip install "apache-airflow[celery,postgres,amazon]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

### Initiate database (Execute Once)

```sh
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-01/airflow_home"
airflow db migrate
airflow users create --username admin --password admin123 --firstname admin --lastname admin --role Admin --email admin@admin.com
```

### (Optional) Logging Storage: S3 connection

```sh
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-01/airflow_home"
airflow connections \
add 'minio' \
--conn-type 'aws' \
--conn-login 'minioadmin' \
--conn-password 'minioadmin' \
--conn-extra '{"endpoint_url": "http://localhost:9000"}'
```

### Start webserver

```sh
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-01/airflow_home"
airflow webserver --port 8080
```

### Start scheduler

> In new terminal

```sh
source ./node-01/airflow/bin/activate
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-01/airflow_home"
airflow scheduler
```

### Start celery worker

> In new terminal

```sh
source ./node-01/airflow/bin/activate
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-01/airflow_home"
airflow celery worker
```

## Airflow: node-02

### Installation

```sh
# python venv
python -m venv ./node-02/airflow
source ./node-02/airflow/bin/activate

# install airflow
AIRFLOW_VERSION=2.10.3
PYTHON_VERSION="$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
pip install "apache-airflow[celery,postgres,amazon]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

### Start webserver

```sh
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-02/airflow_home"
export AIRFLOW__LOGGING__WORKER_LOG_SERVER_PORT=8893
airflow webserver --port 8081
```

### Start scheduler

> In new terminal

```sh
source ./node-02/airflow/bin/activate
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-02/airflow_home"
export AIRFLOW__LOGGING__WORKER_LOG_SERVER_PORT=8893
airflow scheduler
```

### Start celery worker

> In new terminal

```sh
source ./node-02/airflow/bin/activate
export AIRFLOW_HOME="/workspaces/airflow-ha-demo/node-02/airflow_home"
export AIRFLOW__LOGGING__WORKER_LOG_SERVER_PORT=8893
airflow celery worker
```

## HA Option

### haproxy

```sh
docker run \
-d \
--name haproxy \
--hostname haproxy \
--restart=always \
--network=host \
-v $(pwd)/ha/haproxy:/usr/local/etc/haproxy \
haproxy:3.0
```

Goto: http://127.0.0.1:18080/
