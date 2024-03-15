# Debezium

Figure out how to apply configuration for Kafka Connect and the connector itself.

## Links

- Tutorial: <https://debezium.io/documentation/reference/2.5/tutorial.html>
- PostgreSQL runtime configuration: <https://www.postgresql.org/docs/16/runtime-config.html>
- PostgreSQL Debezium connector configuration: <https://debezium.io/documentation/reference/2.5/connectors/postgresql.html>

The tutorial is with MySQL and apparently an example database. Below, however, is with PostgreSQL.

It's a RH project so of course they use their own Zookeeper and Kafka builds :/

## Start everything

    docker-compose up

In the end, we should see something like

```raw
2024-01-26 20:06:19,509 INFO   ||  [Worker clientId=connect-1, groupId=1] Starting connectors and tasks using config offset -1   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2024-01-26 20:06:19,509 INFO   ||  [Worker clientId=connect-1, groupId=1] Finished starting connectors and tasks   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
```

in the logs. Kafka Connect is up and running and we should be able to configure it now through it's REST interface.

## Verify database runtime configuration

    $ psql -h localhost -U postgres -d inventory
    psql (16.1 (Debian 16.1-1.pgdg120+1))
    Type "help" for help.

    inventory=# show wal_level;
     wal_level
    -----------
     logical
    (1 row)

## Add some data

```sql
create table customer (
    id serial primary key,
    first_name varchar(50),
    last_name varchar(50),
    email varchar(100),
    phone varchar(15),
    address varchar(255),
    city varchar(50),
    state varchar(50),
    zip_code varchar(10)
);
```

```sql
insert into customer (first_name, last_name, email, phone, address, city, state, zip_code) values
('John', 'Doe', 'john.doe@example.com', '123-456-7890', '123 Main St', 'Anytown', 'Anystate', '12345'),
('Jane', 'Doe', 'jane.doe@example.com', '987-654-3210', '456 Main St', 'Anytown', 'Anystate', '12345'),
('Jim', 'Smith', 'jim.smith@example.com', '555-555-5555', '789 Main St', 'Anytown', 'Anystate', '12345'),
('Jill', 'Smith', 'jill.smith@example.com', '444-444-4444', '101112 Main St', 'Anytown', 'Anystate', '12345');
```

## Create a connector

Debezium uses an HTTP end-point to accept and modify connector configuration. It is mapped on port 8083.

    $ curl -H "Accept:application/json" localhost:8083/
    {"version":"3.5.1","commit":"2c6fb6c54472e90a","kafka_cluster_id":"pGhKRa65SaKs6hCM1g8StQ"}

No connectors are currently running:

    $ curl -H "Accept:application/json" localhost:8083/connectors/
    []

## Deploy the PostgreSQL connector

For this we need to

- Register the PostgresQL connector to listen to the `inventory` database.
- Watch the PostgreSQL connector start.

This is the connector config: ![postgresql.json](postgresql.json)

 Set it like this:

    $ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @postgresql.json
    HTTP/1.1 201 Created
    Date: Mon, 29 Jan 2024 22:33:16 GMT
    Location: http://localhost:8083/connectors/inventory-connector
    Content-Type: application/json
    Content-Length: 547
    Server: Jetty(9.4.52.v20230823)

    {"name":"inventory-connector","config":{"connector.class":"io.debezium.connector.postgresql.PostgresConnector","tasks.max":"1","plugin.name":"pgoutput","database.hostname":"postgres","database.port":"5432","database.user":"postgres","database.password":"mysecret","database.dbname":"postgres","topic.prefix":"fulfillment","table.include.list":"public.inventory","schema.history.internal.kafka.bootstrap.servers":"kafka:9092","schema.history.internal.kafka.topic":"schema-changes.inventory","name":"inventory-connector"},"tasks":[],"type":"source"}

If there are no errors in the config, you should get back `201 Created` here.

Verify that inventory-connector is included in the list of connectors:

    $ curl -H "Accept:application/json" localhost:8083/connectors/
    ["inventory-connector"]

Review the connector's tasks:

    $ curl -s -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector | jq
    {
      "name": "inventory-connector",
      "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.user": "postgres",
        "database.dbname": "inventory",
        "tasks.max": "1",
        "schema.history.internal.kafka.bootstrap.servers": "broker:29092",
        "plugin.name": "pgoutput",
        "database.port": "5432",
        "topic.prefix": "fulfillment",
        "schema.history.internal.kafka.topic": "schema-changes.inventory",
        "database.hostname": "postgres",
        "database.password": "mysecret",
        "name": "inventory-connector",
        "table.include.list": "public.customer"
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

Remember that a connector is creating state on the broker. Restarting the Kafka Connect container or pod does not remove the connector. Remove a connector by referring to it's name:

    $ curl -i -X DELETE localhost:8083/connectors/inventory-connector/
    HTTP/1.1 204 No Content
    Date: Mon, 29 Jan 2024 21:13:50 GMT
    Server: Jetty(9.4.52.v20230823)

## Watch for changes in topic

List topics created so far:

    $ docker exec broker /bin/sh -c '/bin/kafka-topics --list --bootstrap-server localhost:29092'
    __consumer_offsets
    fulfillment.public.customer
    my_connect_configs
    my_connect_offsets
    my_connect_statuses

Consume:

    $ docker exec broker /bin/sh -c '/bin/kafka-console-consumer --bootstrap-server localhost:29092 --topic fulfillment.public.customer --partition 0 --offset earliest' | jq
    {
      "id": 1,
      "first_name": null,
      "last_name": null,
      "email": null,
      "phone": null,
      "address": null,
      "city": null,
      "state": null,
      "zip_code": null,
      "__deleted": "true"
    }
    {
      "id": 5,
      "first_name": "John",
      "last_name": "Doe",
      "email": "john.doe@example.com",
      "phone": "123-456-7890",
      "address": "123 Main St",
      "city": "Anytown",
      "state": "Anystate",
      "zip_code": "12345",
      "__deleted": "false"
    }
    {
      "id": 6,
      "first_name": "Jane",
      "last_name": "Doe",
      "email": "jane.doe@example.com",
      "phone": "987-654-3210",
      "address": "456 Main St",
      "city": "Anytown",
      "state": "Anystate",
      "zip_code": "12345",
      "__deleted": "false"
    }
    {
      "id": 7,
      "first_name": "Jim",
      "last_name": "Smith",
      "email": "jim.smith@example.com",
      "phone": "555-555-5555",
      "address": "789 Main St",
      "city": "Anytown",
      "state": "Anystate",
      "zip_code": "12345",
      "__deleted": "false"
    }
    {
      "id": 8,
      "first_name": "Jill",
      "last_name": "Smith",
      "email": "jill.smith@example.com",
      "phone": "444-444-4444",
      "address": "101112 Main St",
      "city": "Anytown",
      "state": "Anystate",
      "zip_code": "12345",
      "__deleted": "false"
    }

From here on changes are captured and produced to the topic. Leave the terminal open. In a new terminal remove a row. You see a delete message in the topic.

    inventory=# delete from customer where id = 1;
    DELETE 1

This will create a message:

    {
      "id": 1,
      "first_name": null,
      "last_name": null,
      "email": null,
      "phone": null,
      "address": null,
      "city": null,
      "state": null,
      "zip_code": null,
      "__deleted": "true"
    }

If you want to trigger a change message for all rows in a table again, you can do something like this now:

    inventory=# update customer set first_name = first_name;
    UPDATE 4

## Further reading

If you want to make this Debezium thing run as a source connector that pushes messages to a broker running on Confluent, you might find this important:
<https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_plain.html#sasl-plain-connect-workers>
You basically have to Connect workers to use `SASL/PLAIN`. That means, for a Debezium source connector you have to configure the same properties adding the `producer` prefix to the worker config.

## Next steps

Get this working with a Kafka broker running on Confluent cloud.
