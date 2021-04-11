# ELK Resilience Example

This repository contains a demo of an problems observed within a
running system. Two instances of the system were deployed in different
regions for resilience and to maintain centralised logging all log
messages from each cluster are routed to to elastic instances in
**both** regions. e.g.

```
output {
    elasticsearch {
        hosts => "http://elastic1:9200"
    }
    elasticsearch {
        hosts => "http://elastic2:9200"
    }
}
```

It was discovered that (as documented) if the first output fails, then
the pipeline stops. This means that, as far as resilience goes, once
`elastic1` is down there is **no logging**.

One solution to this appears to be the [output isolator pattern](https://www.elastic.co/guide/en/logstash/current/pipeline-to-pipeline.html#output-isolator-pattern).

## Demo of problem

This repository contains a `docker-compose.yml` that will start up:

1. A pair of elastic+kibana instances. The kibana UI is available on http://localhost:5601 and http://localhost:5602
2. A logstash instance configures with an http input and outputs to each of the elastic instances.

Start the system:

```
docker-compose up -d
```

In one terminal follow the output from logstash:

```
docker-compose logs -f logstash
```

In another terminal send messages to the logstash http input, e.g.:

```
curl http://localhost:31311/an/example/message
```

You should see output to the logstash logs similar to:

```
logstash_1  | {
logstash_1  |     "@timestamp" => 2021-04-11T09:35:35.967Z,
logstash_1  |       "@version" => "1",
logstash_1  |        "headers" => {
logstash_1  |          "content_length" => "0",
logstash_1  |            "request_path" => "/an/example/message",
logstash_1  |          "request_method" => "GET",
logstash_1  |         "http_user_agent" => "curl/7.64.1",
logstash_1  |               "http_host" => "localhost:31311",
logstash_1  |             "http_accept" => "*/*",
logstash_1  |            "http_version" => "HTTP/1.1"
logstash_1  |     },
logstash_1  |        "message" => "",
logstash_1  |           "host" => "192.168.0.1"
logstash_1  | }
```

If you open up the Kibana UI's (and configure index patterns for the `logstash` alias) you will be able to see data appearing in both instances.

Now stop the first elastic instance and send another message to logstash:

```
docker-compose stop elastic1
curl http://localhost:31311/stopped/elastic1
```

The logs will now have entires like:

```
logstash_1  | {
logstash_1  |     "@timestamp" => 2021-04-11T09:37:33.938Z,
logstash_1  |       "@version" => "1",
logstash_1  |        "headers" => {
logstash_1  |          "content_length" => "0",
logstash_1  |            "request_path" => "/stopped/elastic1",
logstash_1  |          "request_method" => "GET",
logstash_1  |         "http_user_agent" => "curl/7.64.1",
logstash_1  |               "http_host" => "localhost:31311",
logstash_1  |             "http_accept" => "*/*",
logstash_1  |            "http_version" => "HTTP/1.1"
logstash_1  |     },
logstash_1  |        "message" => "",
logstash_1  |           "host" => "192.168.0.1"
logstash_1  | }
logstash_1  | [2021-04-11T09:37:34,050][WARN ][logstash.outputs.elasticsearch][my-pipeline_1][44c348bbda09c5d53b2967f635f5edb8c372f627f5f20192f6016c431cd7ddb0] Marking url as dead. Last error: [LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError] Elasticsearch Unreachable: [http://elastic1:9200/][Manticore::ResolutionFailure] elastic1: Name or service not known {:url=>http://elastic1:9200/, :error_message=>"Elasticsearch Unreachable: [http://elastic1:9200/][Manticore::ResolutionFailure] elastic1: Name or service not known", :error_class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError"}
logstash_1  | [2021-04-11T09:37:34,051][ERROR][logstash.outputs.elasticsearch][my-pipeline_1][44c348bbda09c5d53b2967f635f5edb8c372f627f5f20192f6016c431cd7ddb0] Attempted to send a bulk request to elasticsearch' but Elasticsearch appears to be unreachable or down! {:error_message=>"Elasticsearch Unreachable: [http://elastic1:9200/][Manticore::ResolutionFailure] elastic1: Name or service not known", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError", :will_retry_in_seconds=>2}
logstash_1  | [2021-04-11T09:37:34,559][WARN ][logstash.outputs.elasticsearch][my-pipeline_1] Attempted to resurrect connection to dead ES instance, but got an error. {:url=>"http://elastic1:9200/", :error_type=>LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError, :error=>"Elasticsearch Unreachable: [http://elastic1:9200/][Manticore::ResolutionFailure] elastic1"}
```

_And_ there will be no data published to `elastic2`.

Stop the errors by restarting `elastic1` e.g.

```
docker-compose start elastic1
```

Eventually you'll have a line in the logstash logs saying
`Restored connection to ES instance` and the data will appear in both
elastic instances.

In this scenario, `logstash` is not losing data, it is queued
in memory, but the data does not get sent anywhere until the pipeline can
continue. New messages for the same pipeline are not processed at all.
If the logstash instance were to die all this queued messages are lost.

## Possible Solution

Using the output isolator pattern should allow each output to buffer
some data even when the pipeline is blocked. (**Note** This requires logtash >= 7.0 and that you use a `pipelines.yml` file.)

To try this out examine
`input-pipeline.logstash.conf`.
The contents should currently look like:

```
input {
  http {
    host => "logstash" # default: 0.0.0.0
    port => 31311 # default: 8080
  }
}
output {
    stdout {codec => rubydebug}
    elasticsearch {
        hosts => "http://elastic1:9200"
    }
    elasticsearch {
        hosts => "http://elastic2:9200"
    }
    # pipeline {
    #     send_to => ["es1","es2"]
    # }
}
```

Delete the `elasticsearch` output entries and uncomment the pipeline so the file
looks like:

```
input {
  http {
    host => "logstash" # default: 0.0.0.0
    port => 31311 # default: 8080
  }
}
output {
    stdout {codec => rubydebug}
    pipeline {
        send_to => ["es1","es2"]
    }
}
```

`es1` is defined in `es1.logstash.cong` and `es2` is defined in `es2.logstash.conf`.

Now restart logstash by running:

```
docker-compose restart logstash
```

If you use curl to send messages to the logstash the data should appear in elastic as before. But if you stop `elastic1` and then send a message you will
see data does appear in the `elastic2` instance.

```
docker-compose stop elastic1
curl http://localhost:31311/stopped/elastic1/but/with/pipelines
```

Once `elastic1` is restarted the messages appears there.

# Notes on persistent queues

The configuration in `pipelines.yml` makes the queues for sending on to
each elastic instance persitent, rather than just stored in memory. Note
that the data is not backd up in a distributed manner, so while it will
cope with logstash being restarted it will not survive a full system
failure. Further notes, including the default configuration, can be
found in the [logstash documentation](https://www.elastic.co/guide/en/logstash/current/persistent-queues.html#persistent-queues).

A key consideration is that once the persitent queue is blocked then the
input queue will also become blocked and we are back to the position of
data not making it into either elastic instance. Sizing of persistent
queues should be sufficient to handle expected outages. Reconfiguring
the pipelines may still be required for unexpectedly long outages - but
at least the isolation provides a grace period rather than all logging immedately ceasing.
