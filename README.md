# Kafka

[![Build Status](https://travis-ci.org/dfdx/Kafka.jl.svg?branch=master)](https://travis-ci.org/dfdx/Kafka.jl)

Client for Apache Kafka in Julia

## Status

Basically, Kafka provides 4 principal APIs:

 1. Metadata retrieval 
 2. Producing messages
 3. Fetching messages
 4. Offset management (a.k.a. consumer group management)

First 3 are implemented and should be sufficient for most real-life use cases. The last one, however, is somewhat fast-moving target without single approach (e.g. Kafka 0.8.x uses Zookeeper to store offsets, 0.9.x provides broker API, while external systems like Apache Spark and Apache Storm use their own means to store offsets). Given instability and variety of options this part is postponed for now. Though, proposals on discussions are heavily welcome.

## Usage example

Copied from `examples/all.jl` (see the file for latest version):

```

# the following code assumes you have a Kafka broker running 127.0.0.1:9092
# with topic "test" and one partition

using Kafka

# create KafkaClient using single bootstrap broker
# KafkaClient will then fetch information of all other brokers
# and initialize connections to all of them
kc = KafkaClient("127.0.0.1", 9092)

# get metadata about a topic(s)
# all requests in Kafka.jl return channels that hold actual result
# this way requests may be chained asynchronously and results taken later
md_channel = metadata(kc, ["test"])
md = take!(md_channel)
# or, if you prefer synchronous logic, use one-linear
take!(metadata(kc, ["test"]))
# if you don't pass any topics, metadata for all of them will be returned
take!(metadata(kc))

# produce new messages
# each message is a key-value pair where both key and value are byte arrays
keys = [convert(Vector{UInt8}, key) for key in ["1", "2", "3"]]
values = [convert(Vector{UInt8}, value) for value in ["feel", "good", "inc."]]
messages = collect(zip(keys, values))
# messages are produced to a specific topic (e.g. "test") and partition (e.g. 0)
# take!(produce(...)) returns an offset of the first message
offset = take!(produce(kc, "test", 0, messages))

# fetch messages
# in addition to topic name and partition id, fetch() accepts
# offset to start reading from
# return value is an array of triples (offset, key, value)
# note that due to storage internals, brokers may actually return messages
# with offsets less than start_offset; client is responsible to filter them out
start_offset = 0
offset_messages = take!(fetch(kc, "test", 0, start_offset))
# fetch() supports several options to control troughput/latency tradeoff:
# * max_wait_time - max time to wait for new messages (in milliseconds);
#   default is 100
# * min_bytes - minimum number of bytes that broker should have before returning
#   response to a client; default is 1KB
# * max_bytes - maximum number of bytes to include into response;
#   default is 1M
offset_messages = take!(fetch(kc, "test", 0, start_offset, max_wait_time=5000))

# Kafka.jl tries to make convenient wrapper around Kafka protocol,
# but you can always get access to raw responses using methods with "_" prefix
md_resp = take!(_metadata(kc, ["test"]))
produce_resp = take!(_produce(kc, "test", 0, messages))
fetch_resp = take!(_fetch(kc, "test", 0, start_offset))
```
