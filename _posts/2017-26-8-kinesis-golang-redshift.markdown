---
layout: post
title:  "Tracking system using Golang + Kinesis + Redshift"
date:   2017-8-26 08:43:59
author: Marc Cobos
categories: Golang
tags:	golang kinesis redshift aws
cover:  "/assets/instacode.png"
thumbnail: "/assets/golang.png"
---

Track how the user uses our app/web/product is one of the basic things that all company should do.
It's possible to track everything you can imagine. Data driven Companies uses this data to improve their products and launch new features.

Some examples:
1. `App has been started`
1. `User change to page 2`
1. `Searcher has been used`
1. `Change of password`

...

In my current job we would like to develop a real time tracking system to track all the events that the product people wants to check to make the best product decisions. I was thinking to do it using Go because it is fast, concurrent and easy to understant and learn. As we are using AWS we decided to investigate a little bit if we can use Kinesis and Redshift instead of Kafka and Hive. And this is a small script that we did to check if our tracking system can be done using this technologies.

Kinesis is used to collect and process large streams of data records in real time. And Redshift is a fully managed, petabyte-scale data warehouse service in the cloud.

### Kinesis + Redshift + Golang
For this test we are going to use a Kinesis with one stream + Golang to put data into the stream and consume this data to persist it in Redshift.

<a href="{{ site.baseurl }}/assets/kinesis.png" data-lightbox="Kinesis" data-title="Kinesis">
  <img style="width: 100%; margin:auto;" src="{{ site.baseurl }}/assets/kinesis.png" class="rounded_big" title="Kinesis">
</a>

##### Golang code to put events in the Kinesis stream
```
sess := session.Must(session.NewSession())

// Create a Firehose client with additional configuration
firehoseService := firehose.New(sess, aws.NewConfig().WithRegion("us-east-1"))

recordsBatchInput := &firehose.PutRecordBatchInput{}
recordsBatchInput = recordsBatchInput.SetDeliveryStreamName(streamName)

records := []*firehose.Record{}

for i := 0; i < 10; i++ {
  data := FakeEntity{
    ID:          rand.Intn(maxUint),
    Name:        fmt.Sprintf("Name %d", rand.Intn(maxUint)),
    Description: fmt.Sprintf("Test %d", rand.Intn(maxUint)),
  }

  b, _ := json.Marshal(data)

  record := &firehose.Record{Data: b}
  records = append(records, record)
}

recordsBatchInput = recordsBatchInput.SetRecords(records)

resp, err := firehoseService.PutRecordBatch(recordsBatchInput)
if err != nil {
  fmt.Printf("PutRecordBatch err: %v\n", err)
} else {
  fmt.Printf("PutRecordBatch: %v\n", resp)
}
```

##### Golang kinesis consumer that stores data to redshift

```
describeStreamOutput, err := kinesisService.DescribeStream(&kinesis.DescribeStreamInput{StreamName: &streamName})
if err != nil {
    fmt.Printf("DescribeStream err: %v\n", err)
    os.Exit(1)
}

wg := sync.WaitGroup{}

for _, shard := range describeStreamOutput.StreamDescription.Shards {
    wg.Add(1)
    go getRecordsFromShard(kinesisService, &streamName, shard, &wg, redshift)
}

wg.Wait()
```

Each goroutine executes the following function to read from the shard and store the events in Redshift.

```
func getRecordsFromShard(kinesisService *kinesis.Kinesis, streamName *string, shard *kinesis.Shard, wg *sync.WaitGroup, redshift *sql.DB) {
    defer wg.Done()

    shardIteratorTypeTimestamp := kinesis.ShardIteratorTypeAtTimestamp
    shardIteratorTypeSequenceNumber := kinesis.ShardIteratorTypeAfterSequenceNumber

    timestamp := time.Now()
    timestamp = timestamp.Add(-5 * time.Minute)
    shardIteratorInput := &kinesis.GetShardIteratorInput{
        ShardId:           shard.ShardId,
        StreamName:        streamName,
        ShardIteratorType: &shardIteratorTypeTimestamp,
        Timestamp:         &timestamp,
    }

    shardIteratorOutput, err := kinesisService.GetShardIterator(shardIteratorInput)
    if err != nil {
        return
    }

    query := fmt.Sprintf("INSERT INTO kinesis_test (name, description, id, server_timestamp) VALUES")

    queryValues := []string{}
    sequenceNumber := shard.SequenceNumberRange.StartingSequenceNumber
    limitGetRecords := int64(2)
    for {
        kinesisInput := &kinesis.GetRecordsInput{
            Limit:         &limitGetRecords,
            ShardIterator: shardIteratorOutput.ShardIterator,
        }

        recordsOutput, err := kinesisService.GetRecords(kinesisInput)
        if err != nil {
            log.Printf("ShardID %s, GetRecords err: %v\n", *shard.ShardId, err)
            return
        }

        if len(recordsOutput.Records) > 0 {
            for _, d := range recordsOutput.Records {
                var fakeEntity FakeEntity

                err := json.Unmarshal(d.Data, &fakeEntity)
                if err != nil {
                    log.Printf("GetRecords Unmarshal err: %v\n", err)
                    return
                }

                queryValues = append(queryValues, fmt.Sprintf("('%s', '%s', %d, '%s')", fakeEntity.Name, fakeEntity.Description, fakeEntity.ID, time.Now().UTC().Format("2006-01-02T15:04:05-0700")))
                sequenceNumber = d.SequenceNumber
            }
        } else {
            break
        }

        shardIteratorInput.StartingSequenceNumber = sequenceNumber
        shardIteratorInput.ShardIteratorType = &shardIteratorTypeSequenceNumber
        shardIteratorOutput, err = kinesisService.GetShardIterator(shardIteratorInput)
        if err != nil {
            log.Printf("ShardID %s, GetShardIterator err: %v\n", *shard.ShardId, err)
            return
        }
    }

    var insetsStatement string
    for i := 1; i < len(queryValues); i++ {
        if i == 1 {
            insetsStatement = queryValues[i-1]
        }
        insetsStatement = fmt.Sprintf("%s, %s", insetsStatement, queryValues[i])
    }

    query = fmt.Sprintf("%s %s;", query, insetsStatement)
    _, err = redshift.Exec(query)
    if err != nil {
        return
    }
}
```

After to execute this script we saw that we are able to use this technologies to develop our tracking system.

Gist with publisher code: [Github gist with examples][github gist]


#### Updated Agust 2018:
After some time with our tracking system using Kinesis, Redshift and Go we can say that our tracking system is working perfect with more than 400k users and more that 1TB of events stored in real time with almost no problems.

Here you can see the num of events processed by our system each minute.


<a href="{{ site.baseurl }}/assets/tracking_kinesis.png" data-lightbox="Kinesis" data-title="Kinesis">
  <img style="width: 100%; margin:auto;" src="{{ site.baseurl }}/assets/tracking_kinesis.png" title="Kinesis">
</a>

[github gist]: https://gist.github.com/coboshm/1c89bcc7bf2c9f9694e4984051474951
[github gist consumer]: https://gist.github.com/coboshm/1c89bcc7bf2c9f9694e4984051474951
