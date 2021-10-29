# kafka-sample-producer

This repository shows you how to connect to one or more Kafka topics to stream data into Deephaven. It uses an example producer, which connects to a DevExperts DXFeed demo feed and publishes its quote and trade events as JSON messages through Kafka.

The demo feed contains a handful of symbols with 15 minute delayed publication during trading hours. In order to provide events during non-trading hours, the demo feed will replay random old events every few seconds.

The Kafka producer used in this guide creates `quotes` and `trades` topics using the local system as the broker.

## Use the sample Kafka producer container

To launch the latest release, you can clone the repository via:

```shell
git clone https://github.com/deephaven-examples/kafka-sample-producer.git
cd kafka-sample-producer
```

A Docker container containing a Confluent Kafka Community Edition broker and the DXFeed Kafka producer is available in this repository as: `kafka_sample_producer.tar.gz`.

To load this container, inside the repository directory run:

```shell
docker load < kafka_sample_producer.tar.gz
```

Next, one needs to run the broker. To do this you need to list your IP address. On a Mac the command is:

```shell
ipconfig getifaddr en0
```

To run it, execute:

```shell
docker run -e HOSTIP=<IP_address> -dp 9092:9092 ghcr.io/kafka/sample_producer:latest
```

- `IP_address` is an address of your Docker host that is reachable from your Deephaven Community Core container.
- `9092:9092` exposes port 9092 (the default port for Kafka) from the sample producer container to the host's IP.
- `-d` indicates to run disconnected, rather than echoing container output back to the Docker host's console.

In the examples above, this container was started with `HOSTIP=192.168.86.155`, and the queries were then able to connect to it to obtain the `quotes` and `trades` topics.

## Deephaven

This app runs using Deephaven with Docker. See our [Quickstart](https://deephaven.io/core/docs/tutorials/quickstart/) for more information.

To run:

```shell
docker-compose pull
docker-compose up -d
```

Navigate to [http://localhost:10000/ide](http://localhost:10000/ide/) for the Deephaven IDE.

### Create the `quotes` table

The following query creates the `quotes` table:

```python skip-test
from deephaven import KafkaTools
from deephaven import Types as dh
quotes = KafkaTools.consumeToTable({ 'bootstrap.servers' : '10.128.0.252:9092' },
                    'quotes',
                    key=KafkaTools.IGNORE,
                    value=KafkaTools.json([  ('Sym', dh.string),
                                ('AskSize',  dh.int_),
                                ('AskPrice',  dh.double),
                                ('BidSize',  dh.int_),
                                ('BidPrice',  dh.double),
                                ('AskExchange', dh.string),
                                ('BidExchange', dh.string),
                                ('AskTime', dh.long_),
                                ('BidTime', dh.long_)]),
                    table_type='append') \
    .updateView("AskTime = millisToTime(AskTime)") \
    .updateView("BidTime = millisToTime(BidTime)")
```

Let's walk through the query step-by-step:

- [`KafkaTools`](https://deephaven.io/core/javadoc/io/deephaven/kafka/KafkaTools.html) provides the [`consumeToTable`](../reference/data-import-export/Kafka/consumeToTable/) method, which is used to connect to a Kafka broker and receive events.

- The `Types` module, imported as `dh`, contains data type definitions to be used when adding columns/fields for parsing from JSON messages to a Deephaven table. Note the trailing underscore, which is a required part of the name for the int and long types used here.

- `quotes` is the name of the table to create from the [`consumeToTable`](https://deephaven.io/core/docs/reference/data-import-export/Kafka/consumeToTable/) call.

- The [`consumeToTable`](https://deephaven.io/core/docs/reference/data-import-export/Kafka/consumeToTable/) call includes the following:

  - The first argument is a dictionary which can contain Kafka client properties. The one property which we need to set is the `bootstrap.servers` list (the broker to connect to), including its name, or IP address, and port. 9092 is the default port for Kafka.
  - The next argument is the name of the topic to connect to (`quotes`).
  - The next two arguments are the key/value pair to parse from Kafka messages. In this case, we are going to accept all keys (`KafkaTools.IGNORE`), and parse the values as JSON (`KafkaTools.json`), with the array of the `KafkaTools.json` argument being the list of columns/fields and their data types.

- The last two statements ([`.updateView`](https://deephaven.io/core/docs/reference/table-operations/select/update-view/)) operate on the `quotes` table to convert epoch milliseconds Ask and Bid time values to Deephaven `DBDateTime` values.

The result is a `quotes` table which populates as new events arrive:

New messages are added to the bottom of the table, by default, so you may want to [`.reverse()`](https://deephaven.io/core/docs/reference/table-operations/sort/reverse/) it, or [sort in descending order](https://deephaven.io/core/docs/reference/table-operations/sort/sort-descending/) by `KafkaTimestamp`, in order to see new events tick in.

### Create the `trades` table

A similar query gets the `trades` topic data from Kafka:

```python skip-test
from deephaven import KafkaTools
from deephaven import Types as dh
trades = KafkaTools.consumeToTable({ 'bootstrap.servers' : '10.128.0.252:9092' },
                    'trades',
                    key=KafkaTools.IGNORE,
                    value=KafkaTools.json([  ('Sym', dh.string),
                                ('Size',  dh.int_),
                                ('Price',  dh.double),
                                ('DayVolume',  dh.int_),
                                ('Exchange', dh.string),
                                ('Time', dh.long_)]),
                    table_type='append') \
    .updateView("Time = millisToTime(Time)")
```

### Perform aggregations

We can then write other queries which combine and/or aggregate data from these streams. This query uses an [as-of join](https://deephaven.io/core/docs/reference/table-operations/join/aj/) to correlate trade events with the most recent bid for the same symbol.

```python skip-test
relatedQuotes = trades.aj(quotes, "Sym,Time=BidTime", "BidTime,BidPrice,BidSize,BidExchange")
```

The next query calculates the volume average price for each symbol on a per-minute basis, using the start of the minute as the binning value.

```python skip-test
from deephaven import caf
vwap = trades.view("Sym","Size","Price","TimeBin=lowerBin(Time,1*MINUTE)","GrossPrice=Price*Size")\
    .by(caf.AggCombo(caf.AggAvg("AvgPrice = Price"),\
    caf.AggSum("Volume = Size"),\
    caf.AggSum("TotalGross = GrossPrice")), "Sym","TimeBin")\
    .updateView("VWAP=TotalGross/Volume")
```

:::note

The `relatedQuotes` and `vwap` tables do not work well outside of trading hours, because the before/after hours events lack real timestamps.

:::

### Create a simple plot

Now, we'll create the `aaplVwap` table to limit the data to AAPL events, then plot VWAP vs actual price for each trade event. The query below depends on the `vwap` table created in the previous example:

```python skip-test
from deephaven import Plot
aaplVwap = vwap.where("Sym=`AAPL`")
p = Plot.plot("VWAP",aaplVwap,"TimeBin","VWAP")\
    .plot("Price",trades.where("Sym=`AAPL`"),"Time","Price").show()
```

## Related documentation

- [Simple Kafka import](https://deephaven.io/core/docs/how-to-guides/kafka-simple/)
- [Kafka introduction](https://deephaven.io/core/docs/conceptual/kafka-in-deephaven/)
- [How to connect to a Kafka stream](https://deephaven.io/core/docs/how-to-guides/kafka-stream/)
- [Kafka basic terminology](https://deephaven.io/core/docs/conceptual/kafka-basic-terms/)
- [consumeToTable](https://deephaven.io/core/docs/reference/data-import-export/Kafka/consumeToTable/)
