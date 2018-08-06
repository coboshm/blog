---
layout: post
title:  "Golang testing AWS services"
date:   2018-8-06 05:43:59
author: Marc Cobos
categories: Golang
tags:	golang kinesis testing aws
cover:  "/assets/golang_testing.jpg"
thumbnail: "/assets/aws-go.jpg"
---

After to see this post about [Golang testing][toptal testing post] i would like to add some examples. When we are using AWS Services in our workers, endpoints, or services whats happening when we want to test it?

In this post i'm going to explain with an example how to mock a aws service with the `iface` packages able in all the services provided by amazon in `aws-sdk-go`. In this example we are going to mock SQS client to test a service used to get data from a SQS.

All the AWS services have their `iface` package that provides an interface to enable mocking for testing your code.

1. Kinesis Client `github.com/aws/aws-sdk-go/service/kinesis/kinesisiface`
1. SQS Client `github.com/aws/aws-sdk-go/service/sqs/sqsiface`
...

In my current project we have a worker that is working everytime receive a SQS message, this SQS message is added using Cloudwatch events. This service is this:

```
// SubscriptionChecker is the app who check everyday if one subscription is still active.
type SubscriptionChecker struct {
	sqs             sqsiface.SQSAPI
	purchaseRepo    purchase.Repository
	instrumenter    instrumenter.Instrumenter
	playStoreClient *playstore.Client
	appStoreClient  *appstore.Client
	doneChan        chan struct{}
}

// NewSubscriptionChecker constructor
func NewSubscriptionChecker(services *service.Locator, playStoreClient *playstore.Client, sqs sqsiface.SQSAPI) *SubscriptionChecker {
    ...
}
```

As you can see this `SubscriptionChecker` struct is formed by a SQS client and other services. In order to test our service is important to say that this `SQSAPI` is provided by the `sqsiface` package and not by the `sqs` package. This is needed because with the `sqs` package we will not be able to mock it and test it.

To test this service we will need to create a `_test.go` file with the package `worker_test` and in our test we will create everything needed for our struct in order to call the constructor and test it.

The important thing here is how we create the SQS mock. In the following code you can see the mockedSQS struct created in our test package to be able to mock SQS.

```
type mockedSQS struct {
	sqsiface.SQSAPI
	receivedMessagesCounter *int32
}
```

In our test we call the constructor with this mock like you can see here:

```
...
counter1 := int32(0)
mockedSQS := mockedSQS{
	receivedMessagesCounter: &counter1,
}

subscriptionChecker := application.NewSubscriptionChecker(services, clientPlayStore, mockedSQS)
...
```

Ok, we already create a SQS Mock, we create a service that we want to test and now whats next? In our service code we are only using a few functions provided by SQS Client. And this functions are the ones that we are going to need to mock and decide the comportament that we want for our test.

In our case the both functions that we are using are `ReceiveMessage` and `DeleteMessage` and in our test we are going to mock it.

As this is a worker that starts everytime that receive a message we want to mock that receive a message only the first call to ReceiveMessage. Thats the reason why we added the `receivedMessagesCounter` in order to be able to know if it's the first call or not. In the following code we can see the `ReceiveMessage` function mocked.

```
func (m mockedSQS) ReceiveMessage(input *sqs.ReceiveMessageInput) (*sqs.ReceiveMessageOutput, error) {
	if atomic.LoadInt32(m.receivedMessagesCounter) == 1 {
		return &sqs.ReceiveMessageOutput{Messages: []*sqs.Message{}}, errors.New("error test")
	}

	atomic.AddInt32(m.receivedMessagesCounter, 1)
	body := "message"
	var Message sqs.Message
	Message.Body = &body
	return &sqs.ReceiveMessageOutput{Messages: []*sqs.Message{&Message}}, nil
}
```

The other function that we need to mock is `DeleteMessage` but here we don't want to do nothing especial. It's just needed to be able to use this mockedSQS struct in our test.
```
func (m mockedSQS) DeleteMessage(input *sqs.DeleteMessageInput) (*sqs.DeleteMessageOutput, error) {
	return &sqs.DeleteMessageOutput{}, nil
}
```

Once we already have this mocked we are able to test our service as we already know. If not, and you want to learn more about it you can just read this post: [Golang testing][toptal testing post]

Now we are going to test our service:
```
 ⚙ mcobos@MacBook-Pro-de-Marc  ~/go/src/github.com/lerningamessl/api   master  go test github.com/lerningamessl/api/pkg/application
ok  	github.com/lerningamessl/api/pkg/application	4.364s
```

[toptal testing post]: https://www.toptal.com/go/your-introductory-course-to-testing-with-go
