# Clickhouse Cluster

Clickhouse cluster with 2 shards and 2 replicas built with docker-compose.

Not for production use.

## Run

```sh
docker compose up -d
```

## Profiles

- `default` - no password

## Test it

Login to clickhouse01 console (first node's ports are mapped to localhost)

```sh
clickhouse-client -h localhost
```

Or open `clickhouse-client` inside any container

```sh
docker exec -it clickhouse01 clickhouse-client -h localhost
```

Check node in cluster

```sql
SELECT hostName(), getMacro('replica'), getMacro('shard'), currentUser() FROM clusterAllReplicas(test_cluster);
```

Create a test database and table (sharded and replicated)

```sql
CREATE DATABASE company_db ON CLUSTER 'test_cluster';

CREATE TABLE company_db.events ON CLUSTER 'test_cluster' (
    time DateTime,
    uid  Int64,
    type LowCardinality(String)
)
ENGINE = ReplicatedMergeTree
PARTITION BY toDate(time)
ORDER BY (uid);

CREATE TABLE company_db.events_distr ON CLUSTER 'test_cluster'
AS company_db.events
ENGINE = Distributed('test_cluster', company_db, events, rand());
```

Load some data

```sql
INSERT INTO company_db.events_distr VALUES
    ('2020-01-01 10:00:00', 100, 'view'),
    ('2020-01-01 10:05:00', 101, 'view'),
    ('2020-01-01 11:00:00', 100, 'contact'),
    ('2020-01-01 12:10:00', 101, 'view'),
    ('2020-01-02 08:10:00', 100, 'view'),
    ('2020-01-02 13:00:00', 103, 'view'),
    ('2020-01-02 15:00:00', 99,  'view'),
    ('2020-01-02 16:00:00', 66,  'view'),
;
```

Check data from the current shard

```sql
SELECT * FROM company_db.events;
```

Check data from all cluster

```sql
SELECT _shard_num, * FROM company_db.events_distr;
```

## Add more nodes

If you need more Clickhouse nodes, add them like this:

1. Add replicas/shards to `config.xml` to the block `config.d/remote_servers.xml`.
2. Add nodes to `docker-compose.yml`.

## Start, stop

Start/stop the cluster without removing containers

```sh
docker compose start
docker compose stop
```

## Teardown

Stop and remove containers and volume

```sh
docker compose down -v
```
