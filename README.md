[![Build Status][badge-travis]][travis]
[![Test Coverage][badge-coverage]][coverage]
[![bitHound Code][badge-bithound-code]][bithound-code]
[![bitHound Overall Score][badge-bithound-overall]][bithound-overall]
[![bitHound Dependencies][badge-bithound-deps]][bithound-deps]
[![bitHound Dev Dependencies][badge-bithound-dev-deps]][bithound-dev-deps]
[![license][badge-license]][license]

# no-kafka-slim

__no-kafka-slim__ is [Apache Kafka](https://kafka.apache.org) 0.9 client for Node.js with [new unified consumer API](#groupconsumer-new-unified-consumer-api) support. This module is a direct fork of [oleksiyk/kafka](https://github.com/oleksiyk/kafka), but with removed depency for Snappy (due to various problems on Windows).

All methods will return a [promise](https://github.com/petkaantonov/bluebird)

* [Using](#using)
* [Producer](#producer)
  * [Producer options](#producer-options)
* [Simple Consumer](#simpleconsumer)
  * [Simple Consumer options](#simpleconsumer-options)
* [Group Consumer](#groupconsumer-new-unified-consumer-api)
  * [Assignment strategies](#assignment-strategies)
  * [Group Consumer options](#groupconsumer-options)
* [Group Admin](#groupadmin-consumer-groups-api)
* [Compression](#compression)
* [Logging](#logging)
* [License](#license)

## Using

* [download and install Kafka](https://kafka.apache.org/documentation.html#quickstart)
* create your test topic:

```shell
kafka-topics.sh --zookeeper 127.0.0.1:2181 --create --topic kafka-test-topic --partitions 3 --replication-factor 1
```

* install __no-kafka__

```shell
npm install no-kafka
```


## Producer

Example:

```javascript
var Kafka = require('no-kafka');
var producer = new Kafka.Producer();

return producer.init().then(function(){
  return producer.send({
      topic: 'kafka-test-topic',
      partition: 0,
      message: {
          value: 'Hello!'
      }
  });
})
.then(function (result) {
  /*
  [ { topic: 'kafka-test-topic', partition: 0, offset: 353 } ]
  */
});
```

Send and retry if failed 2 times with 100ms delay:

```javascript
return producer.send(messages, {
  retries: {
    attempts: 2,
    delay: 100
  }
});
```

Accumulate messages into single batch until their total size is >= 1024 bytes or 100ms timeout expires (overwrite Producer constructor options):

```javascript
producer.send(messages, {
  batch: {
    size: 1024,
    maxWait: 100
  }
});
producer.send(messages, {
  batch: {
    size: 1024,
    maxWait: 100
  }
});
```

Please note, that if you pass different options to the `send()` method then these messages will be grouped into separate batches:

```javascript
// will be sent in batch 1
producer.send(messages, {
  batch: {
    size: 1024,
    maxWait: 100
  },
  codec: Kafka.COMPRESSION_GZIP
});
```

### Producer options:
* `requiredAcks` - require acknoledgments for produce request. If it is 0 the server will not send any response.  If it is 1 (default), the server will wait the data is written to the local log before sending a response. If it is -1 the server will block until the message is committed by all in sync replicas before sending a response. For any number > 1 the server will block waiting for this number of acknowledgements to occur (but the server will never wait for more acknowledgements than there are in-sync replicas).
* `timeout` - timeout in ms for produce request
* `clientId` - ID of this client, defaults to 'no-kafka-client'
* `connectionString` - comma delimited list of initial brokers list, defaults to '127.0.0.1:9092'
* `partitioner` - function used to determine topic partition for message. If message already specifies a partition, the partitioner won't be used. The partitioner function receives 3 arguments: the topic name, an array with topic partitions, and the message (useful to partition by key, etc.). `partitioner` can be sync or async (return a Promise).
* `retries` - controls number of attempts at delay between them when produce request fails
  * `attempts` - number of total attempts to send the message, defaults to 3
  * `delay` - delay in ms between retries, defaults to 1000
* `codec` - compression codec, one of Kafka.COMPRESSION_NONE, Kafka.COMPRESSION_GZIP
* `batch` - control batching (grouping) of requests
  * `size` - group messages together into single batch until their total size exceeds this value, defaults to 16384 bytes. Set to 0 to disable batching.
  * `maxWait` - send grouped messages after this amount of milliseconds expire even if their total size doesn't exceed `batch.size` yet, defaults to 10ms. Set to 0 to disable batching.
* `asyncCompression` - boolean, use asynchronouse compression instead of synchronous, defaults to `false`

## SimpleConsumer

Manually specify topic, partition and offset when subscribing. Suitable for simple use cases.

Example:

```javascript
var consumer = new Kafka.SimpleConsumer();

// data handler function can return a Promise
var dataHandler = function (messageSet, topic, partition) {
    messageSet.forEach(function (m) {
        console.log(topic, partition, m.offset, m.message.value.toString('utf8'));
    });
};

return consumer.init().then(function () {
    // Subscribe partitons 0 and 1 in a topic:
    return consumer.subscribe('kafka-test-topic', [0, 1], dataHandler);
});
```

Subscribe (or change subscription) to specific offset and limit maximum received MessageSet size:

```javascript
consumer.subscribe('kafka-test-topic', 0, {offset: 20, maxBytes: 30}, dataHandler)
```

Subscribe to latest or earliest offsets in the topic/parition:

```javascript
consumer.subscribe('kafka-test-topic', 0, {time: Kafka.LATEST_OFFSET}, dataHandler)
consumer.subscribe('kafka-test-topic', 0, {time: Kafka.EARLIEST_OFFSET}, dataHandler)
```

Subscribe to all partitions in a topic:

```javascript
consumer.subscribe('kafka-test-topic', dataHandler)
```

Commit offset(s) (V0, Kafka saves these commits to Zookeeper)

```javascript
consumer.commitOffset([
  {
      topic: 'kafka-test-topic',
      partition: 0,
      offset: 1
  },
  {
      topic: 'kafka-test-topic',
      partition: 1,
      offset: 2
  }
])
```

Fetch commited offset(s)

```javascript
consumer.fetchOffset([
  {
      topic: 'kafka-test-topic',
      partition: 0
  },
  {
      topic: 'kafka-test-topic',
      partition: 1
  }
]).then(function (result) {
/*
[ { topic: 'kafka-test-topic',
    partition: 1,
    offset: 2,
    metadata: null,
    error: null },
  { topic: 'kafka-test-topic',
    partition: 0,
    offset: 1,
    metadata: null,
    error: null } ]
*/
});
```

### SimpleConsumer options
* `groupId` - group ID for comitting and fetching offsets. Defaults to 'no-kafka-group-v0'
* `timeout` - timeout for fetch requests, defaults to 100ms
* `idleTimeout` - timeout between fetch calls, defaults to 1000ms
* `minBytes` - minimum number of bytes to wait from Kafka before returning the fetch call, defaults to 1 byte
* `maxBytes` - maximum size of messages in a fetch response
* `clientId` - ID of this client, defaults to 'no-kafka-client'
* `connectionString` - comma delimited list of initial brokers list, defaults to '127.0.0.1:9092'
* `recoveryOffset` - recovery position (time) which will used to recover subscription in case of OffsetOutOfRange error, defaults to Kafka.LATEST_OFFSET
* `asyncCompression` - boolean, use asynchronouse decompression instead of synchronous, defaults to `false`

## GroupConsumer (new unified consumer API)

Specify an assignment strategy (or use __no-kafka__ built-in consistent or round robin assignment strategy) and subscribe by specifying only topics. Elected group leader will automatically assign partitions between all group members.

Example:

```javascript
var Promise = require('bluebird');
var consumer = new Kafka.GroupConsumer();

var dataHandler = function (messageSet, topic, partition) {
    return Promise.map(messageSet, function (m){
        console.log(topic, partition, m.offset, m.message.value.toString('utf8'));
        // commit offset
        return consumer.commitOffset({topic: topic, partition: partition, offset: m.offset, metadata: 'optional'});
    });
};

var strategies = [{
    strategy: 'TestStrategy',
    subscriptions: ['kafka-test-topic'],
    handler: dataHandler
}];

consumer.init(strategies); // all done, now wait for messages in dataHandler
```

### Assignment strategies

__no-kafka__ provides three built-in strategies:
* `Kafka.WeightedRoundRobinAssignment` weighted round robin assignment (based on [wrr-pool](https://github.com/oleksiyk/wrr-pool)).
* `Kafka.ConsistentAssignment` which is based on a consistent [hash ring](https://github.com/3rd-Eden/node-hashring) and so provides consistent assignment across consumers in a group based on supplied `metadata.id` and `metadata.weight` options.
* `Kafka.RoundRobinAssignment` simple assignment strategy (default).

Using `Kafka.WeightedRoundRobinAssignment`:

```javascript
var strategies = {
    strategy: 'TestStrategy',
    subscriptions: ['kafka-test-topic'],
    metadata: {
        weight: 4
    },
    fn: Kafka.WeightedRoundRobinAssignment,
    handler: dataHandler
};
// consumer.init(strategies)....
```

Using `Kafka.ConsistentAssignment`:

```javascript
var strategies = {
    strategy: 'TestStrategy',
    subscriptions: ['kafka-test-topic'],
    metadata: {
        id: process.argv[2] || 'consumer_1',
        weight: 50
    },
    fn: Kafka.ConsistentAssignment,
    handler: dataHandler
};
// consumer.init(strategies)....
```
Note that each consumer in a group should have its own and consistent metadata.id.

Using `Kafka.RoundRobinAssignment` (default in __no-kafka__):

```javascript
var strategies = {
    strategy: 'TestStrategy',
    subscriptions: ['kafka-test-topic'],
    handler: dataHandler
};
// consumer.init(strategies)....
```

You can also write your own assignment strategy function and provide it as `fn` option of the strategy item.

### GroupConsumer options

* `groupId` - group ID for comitting and fetching offsets. Defaults to 'no-kafka-group-v0.9'
* `timeout` - timeout for fetch requests, defaults to 100ms
* `idleTimeout` - timeout between fetch calls, defaults to 1000ms
* `minBytes` - minimum number of bytes to wait from Kafka before returning the fetch call, defaults to 1 byte
* `maxBytes` - maximum size of messages in a fetch response
* `clientId` - ID of this client, defaults to 'no-kafka-client'
* `connectionString` - comma delimited list of initial brokers list, defaults to '127.0.0.1:9092'
* `sessionTimeout` - session timeout in ms, min 6000, max 30000, defaults to `15000`
* `heartbeatTimeout` - delay between heartbeat requests in ms, defaults to `1000`
* `retentionTime` - offset retention time in ms, defaults to 1 day (24 * 3600 * 1000)
* `startingOffset` - starting position (time) when there is no commited offset, defaults to `Kafka.LATEST_OFFSET`
* `recoveryOffset` - recovery position (time) which will used to recover subscription in case of OffsetOutOfRange error, defaults to Kafka.LATEST_OFFSET
* `asyncCompression` - boolean, use asynchronouse decompression instead of synchronous, defaults to `false`

## GroupAdmin (consumer groups API)

Offes two methods:

* `listGroups` - list existing consumer groups
* `describeGroup` - describe existing group by its id

Example:

```javascript
var admin = new Kafka.GroupAdmin();

return admin.init().then(function(){
    return admin.listGroups().then(function(groups){
        // [ { groupId: 'no-kafka-admin-test-group', protocolType: 'consumer' } ]
        return admin.describeGroup('no-kafka-admin-test-group').then(function(group){
            /*
            { error: null,
              groupId: 'no-kafka-admin-test-group',
              state: 'Stable',
              protocolType: 'consumer',
              protocol: 'TestStrategy',
              members:
               [ { memberId: 'group-consumer-82646843-b4b8-4e91-94c9-b4708c8b05e8',
                   clientId: 'group-consumer',
                   clientHost: '/192.168.1.4',
                   version: 0,
                   subscriptions: [ 'kafka-test-topic'],
                   metadata: <Buffer 63 6f 6e 73 75 6d 65 72 2d 6d 65 74 61 64 61 74 61>,
                   memberAssignment:
                    { _blength: 44,
                      version: 0,
                      partitionAssignment:
                       [ { topic: 'kafka-test-topic',
                           partitions: [ 0, 1, 2 ] },
                          ],
                      metadata: null } },
                  ] }
             */
        })
    });
});
```

## Compression

__no-kafka__ supports Gzip compression.

Enable compression in Producer:

```javascript
var Kafka = require('no-kafka');

var producer = new Kafka.Producer({
    clientId: 'producer',
    codec: Kafka.COMPRESSION_GZIP // Kafka.COMPRESSION_NONE, Kafka.COMPRESSION_GZIP
});
```

Alternatively just send some messages with specified compression codec (overwrites codec set in contructor):

```javascript
return producer.send({
    topic: 'kafka-test-topic',
    partition: 0,
    message: { value: 'p00' }
}, { codec: Kafka.COMPRESSION_GZIP })
```

By default __no-kafka__ will use synchronous compression and decompression (synchronous Gzip is not availble in node < 0.11).
Enable async compression/decompression with `asyncCompression` options:

Producer:

```javascript
var producer = new Kafka.Producer({
    clientId: 'producer',
    asyncCompression: true,
    codec: Kafka.COMPRESSION_GZIP
});
```

Consumer:

```javascript
var consumer = new Kafka.SimpleConsumer({
    idleTimeout: 100,
    clientId: 'simple-consumer',
    asyncCompression: true
});
```

## Logging

You can differentiate messages from several instances of producer/consumer by providing unique `clientId` in options:

```javascript
var consumer1 = new Kafka.GroupConsumer({
    clientId: 'group-consumer-1'
});
var consumer2 = new Kafka.GroupConsumer({
    clientId: 'group-consumer-2'
});
```
=>

```
2016-01-12T07:41:57.884Z INFO group-consumer-1 ....
2016-01-12T07:41:57.884Z INFO group-consumer-2 ....
```

Change the logging level:

```javascript
var consumer = new Kafka.GroupConsumer({
    clientId: 'group-consumer',
    logger: {
        logLevel: 1 // 0 - nothing, 1 - just errors, 2 - +warnings, 3 - +info, 4 - +debug, 5 - +trace
    }
});
```

Send log messages to Logstash server(s) via UDP:

```javascript
var consumer = new Kafka.GroupConsumer({
    clientId: 'group-consumer',
    logger: {
        logstash: {
            enabled: true,
            connectionString: '10.0.1.1:9999,10.0.1.2:9999',
            app: 'myApp-kafka-consumer'
        }
    }
});
```

You can overwrite the function that outputs messages to stdout/stderr:

```javascript
var consumer = new Kafka.GroupConsumer({
    clientId: 'group-consumer',
    logger: {
        logFunction: console.log
    }
});
```

## License: [MIT](https://github.com/oleksiyk/kafka/blob/master/LICENSE)

[badge-license]: https://img.shields.io/badge/License-MIT-green.svg
[license]: https://github.com/oleksiyk/kafka/blob/master/LICENSE
[badge-travis]: https://api.travis-ci.org/oleksiyk/kafka.svg?branch=master
[travis]: https://travis-ci.org/oleksiyk/kafka
[badge-coverage]: https://codeclimate.com/github/oleksiyk/kafka/badges/coverage.svg
[coverage]: https://codeclimate.com/github/oleksiyk/kafka/coverage
[badge-david-deps]: https://david-dm.org/oleksiyk/kafka.svg
[david-deps]: https://david-dm.org/oleksiyk/kafka
[badge-david-dev-deps]: https://david-dm.org/oleksiyk/kafka/dev-status.svg
[david-dev-deps]: https://david-dm.org/oleksiyk/kafka#info=devDependencies
[badge-bithound-code]: https://www.bithound.io/github/oleksiyk/kafka/badges/code.svg
[bithound-code]: https://www.bithound.io/github/oleksiyk/kafka
[badge-bithound-overall]: https://www.bithound.io/github/oleksiyk/kafka/badges/score.svg
[bithound-overall]: https://www.bithound.io/github/oleksiyk/kafka
[badge-bithound-deps]: https://www.bithound.io/github/oleksiyk/kafka/badges/dependencies.svg
[bithound-deps]: https://www.bithound.io/github/oleksiyk/kafka/master/dependencies/npm
[badge-bithound-dev-deps]: https://www.bithound.io/github/oleksiyk/kafka/badges/devDependencies.svg
[bithound-dev-deps]: https://www.bithound.io/github/oleksiyk/kafka/master/dependencies/npm
