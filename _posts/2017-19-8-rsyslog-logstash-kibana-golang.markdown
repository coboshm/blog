---
layout: post
title:  "Logging our golang app using rsyslog + logstash + kibana"
date:   2017-8-19 08:43:59
author: Marc Cobos
categories: Golang
tags:	golang rsyslog logstash
cover:  "/assets/instacode.png"
thumbnail: "/assets/logstash-logo.png"
---
Nowadays Elasticsearch is one of the best solutions for our logs. Here you have a configuration used in one of my projects. There is a logstash configuration + rsyslog config + logging in our golang app.


<a href="{{ site.baseurl }}/assets/logstash-elastic.png" data-lightbox="Elasticsearch + logstash + kibana" data-title="Elasticsearch + logstash + kibana">
  <img style="width: 100%; margin:auto;" src="{{ site.baseurl }}/assets/logstash-elastic.png" class="rounded_big" title="Sailbot">
</a>

Repo with all the code: [Github repo with examples][github]

## Logging in our golang application
#### Creating our logger
```
log, err := logger.NewLoggerFromDSN(loggerDSN, appName, *env)
```

The DSN (defined in our `config.toml`) should looks like:
```
dsn = "kibana://?level=debug"
```

Schema could be:
1. kibana `kibana://?level=debug`
1. stdout `stdout://?level=info`
1. discardall `discardall://?level=debug`

Level parameter could be `debug` or `info`

#### Using our logger
```
log.Info("Running...", logger.NewField("newField1", "value1"), logger.NewField("newField2", 2))
```

```
log.Debug("Debugging Running...", logger.NewField("newField1", "value1"), logger.NewField("newField2", 2))
```


## Instaling rsyslog and logstash in (for Red Hat-based systems)
#### Configure Rsyslog `/etc/rsyslog.conf`

Active mmjsonparse it will be used to parse all the `@cee json messages
```
module(load="mmjsonparse")
```

Create rulset that uses `mmjsonparse` to parse the `@cee messages and do action to forward to logstash udp port the json message
```
ruleset(name="remoteAllJsonLog") {
    action(type="mmjsonparse")
    if $parsesuccess == "OK" then {
        action(
            type="omfwd"
            Target="localhost"
            Port="5514"
            Protocol="udp"
            template="allJsonLogTemplate"
        )
    }
    stop
}
```

Define a template to get all json fields of the message
```
template(name="allJsonLogTemplate" type="list") {
    property(name="$!all-json")
}
```

Get all the logs using this udp port and use the ruleset `remoteAllJsonLog`
```
module(load="imudp") # needs to be done just once
input(type="imudp" port="514" ruleset="remoteAllJsonLog")
```

#### Configure logstash `/etc/logstash/config.d/logstash.conf`
Input get messages from port 5514 UDP
```
input {
  udp {
      port => 5514
      codec => json
  }
}
```

Output send messages to elasticsearch
```
output {
    elasticsearch {
        hosts => "[ELASTICSEARCH_HOST]"
    }
}
```


[github]: https://github.com/coboshm/go-logger-syslog