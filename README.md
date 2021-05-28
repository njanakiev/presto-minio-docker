# presto-minio-docker

Minimal example to run Presto with Minio and the Hive standalone metastore on Docker. The data in this tutorial was converted into an [Apache Parquet](https://parquet.apache.org/) file from the famous [Iris data set](https://archive.ics.uci.edu/ml/datasets/iris).

# Installation and Setup

Install [s3cmd](https://s3tools.org/s3cmd) with:

```bash
sudo apt update
sudo apt install -y \
    s3cmd \
    openjdk-11-jre-headless  # Needed for presto-cli
```

Pull and run all services with:

```bash
docker-compose up
```

Configure `s3cmd` with (or use the `minio.s3cfg` configuration):

```bash
s3cmd --config minio.s3cfg --configure
```

Use the following configuration for the `s3cmd` configuration when prompted:

```
Access Key: minio_access_key
Secret Key: minio_secret_key
Default Region [US]:
S3 Endpoint [s3.amazonaws.com]: localhost:9000
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: localhost:9000
Encryption password:
Path to GPG program [/usr/bin/gpg]:
Use HTTPS protocol [Yes]: no
```

To create a bucket and upload data to minio, type:

```bash
s3cmd --config minio.s3cfg \
  mb s3://iris
s3cmd --config minio.s3cfg \
  put data/iris.parq s3://iris/iris_parquet/iris.parq
```
To list all object in all buckets, type:

```bash
s3cmd --config minio.s3cfg la
```

## Access Presto with CLI and Prepare Table

Download [presto cli](https://prestodb.io/docs/current/installation/cli.html) with:

```bash
wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.253/presto-cli-0.253-executable.jar \
  -O presto
chmod +x presto  # Make it executable
```

Create schema and create table with:

```bash
./presto --execute "
CREATE SCHEMA IF NOT EXISTS minio.iris
WITH (location = 's3a://iris/');

CREATE TABLE IF NOT EXISTS minio.iris.iris_parquet (
  sepal_length DOUBLE,
  sepal_width  DOUBLE,
  petal_length DOUBLE,
  petal_width  DOUBLE,
  class        VARCHAR
)
WITH (
  external_location = 's3a://iris/iris_parquet',
  format = 'PARQUET'
);"
```

Query the newly created table with:

```bash
./presto --execute "
SHOW TABLES IN minio.iris;
SELECT * FROM minio.iris.iris_parquet LIMIT 5;"
```

## Importing data from other Connectors

Create new table in PostgreSQL from [TPCDS](https://prestodb.io/docs/current/connector/tpcds.html):

```bash
./presto --execute "
CREATE TABLE postgresql.public.item AS 
  SELECT i_item_id, i_item_desc 
  FROM tpcds.tiny.item;"
```

Create new table in Minio from TPCDS:

```bash
# Create new bucket
s3cmd --config minio.s3cfg mb s3://data

./presto --execute "
CREATE SCHEMA IF NOT EXISTS minio.data 
WITH (location = 's3a://data/');

CREATE TABLE minio.data.item 
WITH (format = 'PARQUET') AS 
  SELECT i_item_id, i_item_desc 
  FROM tpcds.tiny.item;
```

# License 
This project is licensed under the MIT license. See the [LICENSE](LICENSE) for details.
