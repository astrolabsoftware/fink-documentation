# Livestream

_date 22/07/2025_

This manual has been tested for `fink-client` version 8.8. Other versions might work. In case of trouble, send us an email (contact@fink-broker.org) or [open an issue](https://github.com/astrolabsoftware/fink-client/issues).

## Purpose

The livestream service is based on the Fink filters. After each exposure, Fink processes the alerts sent by ZTF and the filters select alerts to be transmitted based on their content. These alerts are sent to the Fink [Apache Kafka](https://kafka.apache.org/) cluster, and substreams are produced (1 filter = 1 substream), identified by their _topic_ name. Each alert pushed is available 7 days in the queue, and consumers can replay streams indefinitely.

As Kafka can be somehow cumbersome, we developed a client to facilitate the stream consuming part for Fink users: [fink-client](https://github.com/astrolabsoftware/fink-client). Users can connect to one or more topics, and new topics can be created via new Fink filters.

## Installation of fink-client

To ease the consuming step, the users are recommended to use the [fink-client](https://github.com/astrolabsoftware/fink-client), which is a wrapper around Apache Kafka. `fink_client` requires a version of Python 3.9+. Documentation to install the client can be found at [services/fink_client](fink_client.md). Note that you need to be registered in order t
o poll data.

For the list of available topics, see [https://fink-broker.readthedocs.io/en/latest/broker/filters/#available-topics](https://fink-broker.readthedocs.io/en/latest/broker/filters/#available-topics).

## First steps: testing the connection

Processed alerts are stored 4 days on our servers, which means if you forget to poll data, you'll be able to retrieve it up to 4 days after emission. This also means on your first connection, you will have 4 days of alert to retrieve. Before you get all of them, let's retrieve the first available alert to check the connection. On a terminal, run the following

```bash
# access help using `fink_consumer -h`
fink_consumer --display -limit 1
```

This will download the first available alert, and print some useful information.  The alert schema is automatically downloaded from the GitHub repo (see the Troubleshooting section if that command does not work). Then the alert is consumed and you'll move to the next alert. Of course, if you want to keep the data, you need to store it. This can be easily done:

```bash
# create a folder to store alerts
mkdir alertDB

# access help using `fink_consumer -h`
fink_consumer --display --save -outdir alertDB -limit 1
```

This will download the next available alert, display some useful information on screen, and save it (Apache Avro format) on disk. Then if all works, then you can remove the limit, and let the consumer run for ever!

```bash
# access help using `fink_consumer -h`
fink_consumer --display --save -outdir alertDB
```

## Inspecting alerts

Once alerts are saved, you can open it and explore the content. We wrote a small utility to quickly visualise it:

```bash
# access help using `fink_alert_viewer -h`
# Adapt the filename accordingly -- it is <objectId>_<candid>.avro
fink_alert_viewer -filename alertDB/ZTF21aaqkqwq_1549473362115015004.avro
```

of course, you can develop your own tools based on this one! Note Apache Avro is not something supported by default in Pandas for example, so we provide a small utilities to load alerts more easily:

```python
from fink_client.avro_utils import AlertReader

# you can also specify one folder with several alerts directly
r = AlertReader('alertDB/ZTF21aaqkqwq_1549473362115015004.avro')

# convert alert to Pandas DataFrame
r.to_pandas()
```

## Managing offsets

### Checking offsets

You might want to check where you are on the different queues, that is retrieving the offsets for each topic that you are polling:

```bash
fink_consumer --display_statistics

Topic [Partition]                                   Committed        Lag
========================================================================
fink_sso_ztf_candidates_ztf  [4]                            1        972
------------------------------------------------------------------------
Total for fink_sso_ztf_candidates_ztf                       1        972
------------------------------------------------------------------------

Topic [Partition]                                   Committed        Lag
========================================================================
------------------------------------------------------------------------
Total for fink_sso_fink_candidates_ztf                      0          2
------------------------------------------------------------------------
```

In this example, I have two topic, `fink_sso_ztf_candidates_ztf` and `fink_sso_fink_candidates_ztf`. 

For the first topic, there is one active partition on the remote Kafka cluster that served data (number `[4]`). I polled 1 alert (`Committed`), and there are `972` remaining alerts to be polled (`Lag`). As there is only one active partition on the remote Kafka cluster, the total is the same (there could be up to 10 active partitions). For the second topic, I did not start polling as `0` alert has been `Committed`.

### Resetting offsets

Sometimes you might want to poll again alerts, that is restarting to poll from the beginning of a queue. For this, you can use:

```bash
fink_consumer --display -start_at earliest
Resetting offsets to BEGINNING
...
assign TopicPartition{topic=fink_sso_fink_candidates_ztf,partition=0,offset=0,leader_epoch=None,error=None}
...
assign TopicPartition{topic=fink_sso_ztf_candidates_ztf,partition=0,offset=0,leader_epoch=None,error=None}
...
# poll restarts at the first offset
```

All your topic partitions will be reset to the starting offset (`0` in this case). Similarly, you can empty all topics, and restarting polling from the last offset:

```bash
fink_consumer --display -start_at latest
...
assign TopicPartition{topic=fink_sso_fink_candidates_ztf,partition=0,offset=0,leader_epoch=None,error=None}
...
assign TopicPartition{topic=fink_sso_fink_candidates_ztf,partition=4,offset=2,leader_epoch=None,error=None}
...
assign TopicPartition{topic=fink_sso_ztf_candidates_ztf,partition=4,offset=973,leader_epoch=None,error=None}
...
No alerts the last 10 seconds
...
```

Empty partitions will have `offset=0`, but others will have their offset to the latest one. The client will then wait for new data to come. Note that the reset will be actually triggered on the next poll. Hence the command `fink_consumer --display_statistics` will not right away display the reset offsets.
This is particularly useful after a bug in the topic (malformed alerts pushed), and you want a fresh restart.

## Write your own consumer

You can write your own consumer to manipulate alerts upon receival. We give an simple example here. Open your favourite editor, and paste the following lines:

```python
""" Poll the Fink servers only once at a time """
from fink_client.consumer import AlertConsumer
from fink_client.configuration import load_credentials

from astropy.time import Time

import time
import tabulate

def poll_single_alert(myconfig, topics) -> None:
    """ Connect to and poll fink servers once.

    Parameters
    ----------
    myconfig: dic
    	python dictionnary containing credentials
    topics: list of str
    	List of string with topic names
    """
    maxtimeout = 5

    # Instantiate a consumer
    consumer = AlertConsumer(topics, myconfig)

    # Poll the servers
    topic, alert, key = consumer.poll(maxtimeout)

    # Analyse output - we just print some values for example
    if topic is not None:
		utc = time.strftime('%Y-%m-%d %H:%M:%S', time.gmtime())
		table = [
			[
		 		Time(alert['candidate']['jd'], format='jd').iso,
		 		utc,
		 		topic,
		 		alert['objectId'],
		 		alert['cdsxmatch'],
		 		alert['candidate']['magpsf']
		 	],
		]
		headers = [
			'Emitted at (UTC)',
			'Received at (UTC)',
			'Topic',
			'objectId',
			'Simbad',
			'Magnitude'
		]
		print(tabulate(table, headers, tablefmt="pretty"))
    else:
        print(
            'No alerts received in the last {} seconds'.format(
                maxtimeout
            )
        )

    # Close the connection to the servers
    consumer.close()


if __name__ == "__main__":
    """ Poll the servers only once at a time """

    # to fill
    myconfig = {
        'bootstrap.servers': '',
        'group_id': ''
    }

    topics = ['', '']

    poll_single_alert(myconfig, topics)
```

You only need to update the `myconfig` dictionnary with the connection information sent to you privately, and the `topics` list with the topics you want to access. Save the file, and in a terminal walk to where the file has been saved and execute it:

```bash
python my_consumer.py
```

You should start to see alert flowing! Dummy example:

```bash
+----------------------------------+---------------------+-------------+--------------+-----------------+--------------------+
|         Emitted at (UTC)         |  Received at (UTC)  |    Topic    |   objectId   |     Simbad      |     Magnitude      |
+----------------------------------+---------------------+-------------+--------------+-----------------+--------------------+
| 2021-11-22 08:33:05.999045+00:00 | 2022-02-09 10:32:51 | test_stream | ZTF17aabvtfi | Candidate_TTau* | 18.799415588378906 |
+----------------------------------+---------------------+-------------+--------------+-----------------+--------------------+
```

When there is no more alerts available upstream, you will start to see:

```bash
# X depends on the timeout you defined in the registration
No alerts the last X seconds
```

Now it is your turn to modify this script to do something meaningful with alerts coming to you!

## Troubleshooting

In case of trouble, send us an email (contact@fink-broker.org) or open an issue (https://github.com/astrolabsoftware/fink-client).

### Wrong schema

A typical error though would be:

```
Traceback (most recent call last):
  File "/Users/julien/anaconda3/bin/fink_consumer", line 10, in <module>
    sys.exit(main())
  File "/Users/julien/Documents/workspace/myrepos/fink-client/fink_client/scripts/fink_consumer.py", line 92, in main
    topic, alert = consumer.poll(timeout=maxtimeout)
  File "/Users/julien/Documents/workspace/myrepos/fink-client/fink_client/consumer.py", line 94, in poll
    alert = _decode_avro_alert(avro_alert, self._parsed_schema)
  File "/Users/julien/Documents/workspace/myrepos/fink-client/fink_client/avroUtils.py", line 381, in _decode_avro_alert
    return fastavro.schemaless_reader(avro_alert, schema)
  File "fastavro/_read.pyx", line 835, in fastavro._read.schemaless_reader
  File "fastavro/_read.pyx", line 846, in fastavro._read.schemaless_reader
  File "fastavro/_read.pyx", line 561, in fastavro._read._read_data
  File "fastavro/_read.pyx", line 456, in fastavro._read.read_record
  File "fastavro/_read.pyx", line 559, in fastavro._read._read_data
  File "fastavro/_read.pyx", line 431, in fastavro._read.read_union
  File "fastavro/_read.pyx", line 555, in fastavro._read._read_data
  File "fastavro/_read.pyx", line 349, in fastavro._read.read_array
  File "fastavro/_read.pyx", line 561, in fastavro._read._read_data
  File "fastavro/_read.pyx", line 456, in fastavro._read.read_record
  File "fastavro/_read.pyx", line 559, in fastavro._read._read_data
  File "fastavro/_read.pyx", line 405, in fastavro._read.read_union
IndexError: list index out of range
```

This error happens when the schema to decode the alert is not matching the alert content. Usually this should not happen (schema is included in the alert payload). In case it happens though, you can force a schema:

```
fink_consumer [...] -schema [path_to_a_good_schema]
```

In case you do not have replacement schemas, you can save the current (faulty) schema that is contained within an alert packet:

```bash
fink_consumer -limit 1 --dump_schema
```

You will see the traceback above, with the message:

```
Schema saved as schema_2024-06-03T11:12:36.855544+00:00.json
```


Then you can inspect the schema manually, or open an issue on the fink-client repository by attaching this schema to your message.

### Authentication error

If you try to poll the servers and get:

```
%3|1634555965.502|FAIL|rdkafka#consumer-1| [thrd:sasl_plaintext://xx.xx.xx.xx:yy/bootstrap]: sasl_plaintext://xx.xx.xx.xx:yy/bootstrap: SASL SCRAM-SHA-512 mechanism handshake failed: Broker: Request not valid in current SASL state: broker's supported mechanisms:  (after 18ms in state AUTH_HANDSHAKE)
```

You are likely giving a password when instantiating the consumer. Check your `~/.finkclient/credentials.yml`, it should contain

```yml
password: null
```

or directly in your code:

```python
# myconfig is a dict that should NOT have
# a 'password' key set
consumer = AlertConsumer(mytopics, myconfig)
```

However, if you want the old behaviour, then you need to specify it using `sasl.*` parameters:

```python
myconfig['sasl.username'] = 'your_username'
myconfig['sasl.password'] = None
consumer = AlertConsumer(mytopics, myconfig)
```

### Timeout error

If you get frequent timeouts while you know there are alerts to poll, try to increase the timeout (in seconds) in your configuration file:

```bash
# edit ~/.finkclient/credentials.yml
maxtimeout: 30
```

<!-- ```bash
# does not work -- how to modify yaml?
fink_client_register -maxtimeout 30
``` -->
