# Debezium

Figure out how to apply configuration for Kafka Connect and the connector itself.

## Links

- Tutorial: <https://debezium.io/documentation/reference/2.5/tutorial.html>
- PostgreSQL runtime configuration: <https://www.postgresql.org/docs/16/runtime-config.html>
- PostgreSQL Debezium connector configuration: <https://debezium.io/documentation/reference/2.5/connectors/postgresql.html>

The tutorial is with MySQL and apparently an example database. Below, however, is with PostgreSQL.

It's a RH project so of course they use their own Zookeeper and Kafka builds :/

## Start Zookeeper

    podman pod create --name=dbz --publish "9092,5432,8083"
    podman run -it --rm --name zookeeper --pod dbz quay.io/debezium/zookeeper:2.4

## Start the broker

    podman run -it --rm --name kafka --pod dbz quay.io/debezium/kafka:2.4

## Start database

    podman run -ti --rm --name postgres --pod dbz -v $PWD/my-postgres.conf:/etc/postgresql/postgresql.conf:z \
        -e POSTGRES_PASSWORD=mysecret \
        -e POSTGRES_DB=inventory \
        postgres:16 \
        -c 'config_file=/etc/postgresql/postgresql.conf'

Use below to spawn a container and execute psql inside it:

    $ podman run -ti --rm --name psql --pod dbz postgres:16 sh -c 'psql -h localhost -U postgres -d inventory'
    psql (16.1 (Debian 16.1-1.pgdg120+1))
    Type "help" for help.

    inventory=# show wal_level;
     wal_level
    -----------
     logical
    (1 row)

inventory=#

Now, add some data.

```sql
CREATE TABLE customer (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(15),
    address VARCHAR(255),
    city VARCHAR(50),
    state VARCHAR(50),
    zip_code VARCHAR(10)
);
```

```sql
INSERT INTO customer (first_name, last_name, email, phone, address, city, state, zip_code) VALUES
('John', 'Doe', 'john.doe@example.com', '123-456-7890', '123 Main St', 'Anytown', 'Anystate', '12345'),
('Jane', 'Doe', 'jane.doe@example.com', '987-654-3210', '456 Main St', 'Anytown', 'Anystate', '12345'),
('Jim', 'Smith', 'jim.smith@example.com', '555-555-5555', '789 Main St', 'Anytown', 'Anystate', '12345'),
('Jill', 'Smith', 'jill.smith@example.com', '444-444-4444', '101112 Main St', 'Anytown', 'Anystate', '12345');
```

## Start Kafka Connect

Finally, start Kafka connect.

    podman run -it --rm --name connect --pod dbz \
        -e GROUP_ID=1 \
        -e CONFIG_STORAGE_TOPIC=my_connect_configs \
        -e OFFSET_STORAGE_TOPIC=my_connect_offsets \
        -e STATUS_STORAGE_TOPIC=my_connect_statuses \
        quay.io/debezium/connect:2.4

We should see something like

```raw
2024-01-26 20:06:19,509 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting connectors and tasks using config offset -1   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2024-01-26 20:06:19,509 INFO   ||  [Worker clientId=connect-1, groupId=1] Finished starting connectors and tasks   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
```

in the logs. Kafka Connect is up and running and we should be able to configure it now through it's REST interface. Spawn an interactive shell container inside the pod

    podman run -ti --rm --name prompt --pod dbz fedora sh

and run

    # curl -H "Accept:application/json" localhost:8083/
    {"version":"3.5.1","commit":"2c6fb6c54472e90a","kafka_cluster_id":"pGhKRa65SaKs6hCM1g8StQ"}

No connectors are currently running:

    # curl -H "Accept:application/json" localhost:8083/connectors/
    []

## Deploy the PostgreSQL connector

For this we need to

- Register the PostgresQL connector to listen to the `inventory` database.
- Watch the PostgreSQL connector start.

This is the connector config:

```json
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "plugin.name": "pgoutput",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "mysecret",
    "database.dbname" : "postgres",
    "topic.prefix": "fulfillment",
    "table.include.list": "public.inventory",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.inventory"
  }
}
```

 Set it like this:

    # curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '
        <above config />
    '

If there are no errors in the config, you should get back `201 Created` here.

Verify that inventory-connector is included in the list of connectors:

    # curl -H "Accept:application/json" localhost:8083/connectors/
    ["inventory-connector"]

Review the connector's tasks:

    # dnf install -y jq
    # curl -s -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector | jq
    {
      "name": "inventory-connector",
      "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.user": "postgres",
        "database.dbname": "postgres",
        "topic.prefix": "fulfillment",
        "schema.history.internal.kafka.topic": "schema-changes.inventory",
        "tasks.max": "1",
        "database.hostname": "postgres",
        "database.password": "mysecret",
        "name": "inventory-connector",
        "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
        "table.include.list": "public.inventory",
        "database.port": "5432"
      },
      "tasks": [
        {
          "connector": "inventory-connector",
          "task": 0
        }
      ],
      "type": "source"
    }

## Delete a connector

Remember that a connector is creating state on the broker. Restarting the Kafka Connect container or pod does not remove the connector. Remove a connector with

    # curl -i -X DELETE localhost:8083/connectors/inventory-connector/

, i.e., by referring to it's name.

## Further reading

If you want to make this Debezium thing run as a source connector that pushes messages to a broker running on Confluent, you might find this important:
<https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_plain.html#sasl-plain-connect-workers>
You basically have to Connect workers to use `SASL/PLAIN`. That means, for a Debezium source connector you have to configure the same properties adding the `producer` prefix to the worker config.

## Next steps

Get this working with a Kafka broker running on Confluent cloud.
