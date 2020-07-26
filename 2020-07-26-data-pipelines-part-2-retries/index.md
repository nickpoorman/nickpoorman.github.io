# Data Pipelines: Part 2 - Retries


### A Different Approach

In the last post ["Data Pipelines: Part 1 - Queuing and Messaging Patterns"](/2020-07-26-data-pipelines-part-1-queuing-and-messaging-patterns/), I discussed the various messaging patterns such as pub-sub and push-pull. I also discussed various queueing storage mechanisms such as in-memory and on disk. The last post left off with a solution for reliably sending messages to a webhook that could possibly experience downtime, or have rate limiting requirements. In this post, I will discuss how we can wire a pub-sub technology to a storage engine to solve some of the issues surrounding downtime, rate limiting, backpressure, retry, and dead letter queues. For demonstration purposes, I will continue on with our totaling of page views example used in the previous post in this series.

### NATS as a Message Relay

For the purpose of this post, I will be using a popular open-source messaging system called [NATS](https://nats.io/). Specifically, the code examples will be using the [Go implementation for NATS](https://github.com/nats-io/nats.go) —although there are [implementations in most other popular languages](https://nats.io/download/).

NATS has two main features we will be using. The first is `Publish`, which takes a subject, also known as a topic, in which to publish messages to, and a message payload.

```go
// Simple Publisher
// subject is foo and the paylod is the "Hello World" bytes.
nc.Publish("foo", []byte("Hello World"))
```

The second is feature is `Subscribe`, which takes a subject to listen for messages on and a function to call for each message that is received on that subject.

```go
// Simple Async Subscriber
nc.Subscribe("foo", func(m *nats.Msg) {
    fmt.Printf("Received a message: %s\n", string(m.Data))
})
```

When a message is published in NATS it goes though a NATS server and is relayed to any subscribers listening to that subject. When the subscriber receives a message the function we provide, will be called asynchronously and will include the message that was received. Let's look at some code for the totaling of page views from the previous post in this series.

Below is an example of a consumer responsible for totaling up page views.

```go
// Connect to the NATS server
nc, _ := nats.Connect(nats.DefaultURL)

var totalPageViews int64

// Subscribe to page_view messages from our users.
nc.Subscribe("page_views", func(m *nats.Msg) {
    fmt.Printf("Received a page view from a user: %s\n", string(m.Data))

    // Increment the number of page views.
    atomic.AddInt64(&totalPageViews, 1) // i.e., totalPageViews++
})
```

Next, we'll create an example of what a user creating a page view message might look like.

```go
// Connect to the NATS server
nc, _ := nats.Connect(nats.DefaultURL)

msg := '{"visitTimestamp":"2020-07-14T18:47:13.919Z","page":"/blog/data-pipeline-queuing-and-messaging-patterns","publishedTimestamp":"2020-07-14T08:00:00.000Z"}'

// User publishes a page_view message
nc.Publish("page_views", []byte(msg))
```

In the above example, you can see how our consumer subscribes to a `page_view` subject, and when it receives a message from our producer, we increment the total number of page views.

In reality, if our users are visiting our website, we might have an API that users make requests to from JavaScript. That API would then be responsible for publishing the page view message over NATS.

### Adding Downstream Consumers

As you may recall in the last post of this series, we walked through a solution for dealing with downtime. In our example, we had a webhook which we were required to send messages to. The webhook was controlled by an analytics SaaS company. We needed to guarantee that our messages would eventually make it to the analytics SaaS company in the event that the analytics webhook was offline for a period of time. We solved this by placing a Streaming technology between our producer and the downstream analytics webhook. This allowed us to control the rate at which we sent messages to the webhook and in doing so, allowed us to pause sending messages to the webhook while it was down.

![Diagram of a SaaS Webhook With Streaming](saas-webhook-streaming.svg)

With that solution in place, we don't have to worry about downtime or rate limiting by the SaaS webhook. But that solution has a scaling flaw. What if we need to send our messages to additional SaaS webhooks? Does that mean we need to spin up additional infrastructure for every additional webhook? Some people would say no. Those people might be using a streaming technology such as Kafka that allows users to produce and consume messages on different topics. Let's say for the sake of this post that we don't have a [24-hour SRE team at the ready who can manage our Kafka cluster for us](https://www.google.com/search?hl=en&sxsrf=ALeKk01o5ijLDYNKn5ar_Sk1H6mbkdrRrg%3A1595436486931&ei=xm0YX6CsOITctQaW06jQAg&q=kafka+post+mortem&oq=kafka+post+mor&gs_lcp=CgZwc3ktYWIQAxgAMgUIIRCgATIFCCEQqwIyBQghEKsCOgQIABBHOgQIABBDOgUIABCRAjoFCC4QkQI6CAguELEDEIMBOgsILhCxAxDHARCjAjoFCC4QsQM6CAgAELEDEIMBOgIILjoECC4QQzoCCAA6CAgAELEDEJECOggILhCxAxCRAjoFCAAQsQM6BwgAELEDEEM6BggAEBYQHjoICAAQFhAKEB5QxaoBWITGAWDc0AFoAHADeACAAYsBiAGUC5IBAzcuN5gBAKABAaoBB2d3cy13aXrAAQE&sclient=psy-ab). Maybe we're using a streaming technology that doesn't have a concept of topics or subjects like [AWS Kinesis](https://aws.amazon.com/kinesis/).

Let's think about some of our requirements here.

1. We need the ability to add more webhooks without manually wiring up new infrastructure.
2. We need to ensure the messages eventually make it to our webhooks. Our solution should be resilient to webhook downtime. It should also be able to deal with rate limit requirements a given webhook might have.
3. We want our messages committed to disk. We will most likely have more data than can be reasonably kept in RAM. Also, committing messages to disk gives us better message durability guarantees should the system restart.
4. We don't want the overhead of managing a large replicated service like Kafka.
5. We need acknowledgement capabilities between every producer, consumer, and webhook.

### Acknowledgements

Before we move on, we need to go back to the last post where we talked about acknowledgments. When sending a message from a producer to a consumer, we need a way for the consumer to acknowledge that the message was received and processed. Let's use the case of consumer-2 and SaaS Webhook in the above diagram. When consumer-2 makes an HTTP Request to SaaS Webhook, SaaS Webhook will accept and process the message. Once the message has been processed, SaaS Webhook will respond with an acknowledgment. In the case of HTTP, the acknowledgment is probably an HTTP 200 response code. Internally, we are using pub-sub so our acknowledgments should be similar. A producer will publish a message and a consumer should respond with an acknowledgment once it has processed the message. We can do this quite simply in NATS.

```go
// Responding to a request message
nc.Subscribe("request", func(m *nats.Msg) {
    m.Respond([]byte("The answer is 42."))
})

// Publish a message
nc.Request("request", []byte("What is the answer to the Ultimate Question of Life, the Universe, and Everything?"))
```

NATS makes it easy to [respond to a request](https://docs.nats.io/nats-concepts/reqreply). Any time you make a request, it sends the request to only one subscriber in a [queue group](https://docs.nats.io/nats-concepts/queue). Along with the request, it includes an inbox address where the response should be sent to. The inbox address is simply another subject specifically created to receive a response for that request. Using the `Request` and `Respond` we can satisfy our acknowledgments requirement. We'll also give the consumer 30 seconds to respond with an acknowledgment before the request times out and results in an error.

```go
// Responding to a request message
nc.Subscribe("page_views", func(m *nats.Msg) {
    m.Respond([]byte("Acknowledged"))
})

msg := '{"visitTimestamp":"2020-07-14T18:47:13.919Z","page":"/blog/data-pipeline-queuing-and-messaging-patterns","publishedTimestamp":"2020-07-14T08:00:00.000Z"}'

// Publish a message
acknowledgmentMsg, err := nc.Request("page_views", []byte(msg), 30*time.Second)
```

### Design by Requirements

What if we thought about the problem slightly differently? What if, instead of putting a streaming technology such as Kinesis, which is really an iteration of a high-performance queue, in front of our webhooks, we replaced streaming-2,3,4...n with a single service? Let's design that service by going over our requirements one at a time.

_Requirement #1 - We need the ability to add more webhooks without manually wiring up new infrastructure._

This requirement is something we should be able to solve with our NATS pub-sub technology. Since our pub-sub technology has the concept of subjects, we'll create a subject for each webhook. NATS specifically has the concept of [subject hierarchies](https://docs.nats.io/nats-concepts/subjects). Publishers can use the `.` character to create the subject hierarchy. We can utilize the subject hierarchies to create the subjects `webhooks.saas_webhook_1` and `webhooks.saas_webhook_2`. This way, we can create a single consumer that subscribes to all messages with the `webhooks.*` subject.

```go
// Subscribe to all webhooks messages.
// Note: We could also use "webhooks.>" to match subjects with multiple tokens
//  such as "webhooks.saas_webhook_1.stats".
nc.Subscribe("webhooks.*", func(msg *nats.Msg) {
    switch wh := determineSaasWebhook(msg) {
        case SaasWebhook1:
            sendToSaasWebhook1(msg)
        case SaaSWebhook2:
            sendToSaasWebhook2(msg)
        default:
            log.Printf("unknown webhook: %s", wh)
            // Don't acknowledge.
            return
    }
    msg.Respond([]byte("Acknowledged"))
})
```

_Requirement #2 - We need to ensure the messages eventually make it to our webhooks. Our solution should be resilient to webhook downtime. It should also be able to deal with rate limit requirements a given webhook might have._

Our system should accept messages on a subject as we established above for Requirement #1. The consumer should then forward the message to the webhook endpoint. **When the webhook endpoint doesn't respond after a certain amount of time, we will say the message has timed out and send the message to a [dead letter queue](https://en.wikipedia.org/wiki/Dead_letter_queue).** We could put this logic in the consumer of every webhook, but it might be better to generalize it and put it in the original publishers of the messages. Let's hold onto that thought for now as we go over the rest of the requirements.

_Requirement #3 - We want our messages committed to disk. We will most likely have more data than can be reasonably kept in RAM. Also, committing messages to disk gives us better message durability guarantees should the system restart._

To keep things simple, I'm choosing a persistent and fast key-value database called [Badger](https://github.com/dgraph-io/badger). Badger supports concurrent ACID transactions and serializable snapshot isolation guarantees, plus it's written in Go so we can easily embed it into the service we write with NATS. This will make our lives easier, especially when it comes to testing. With our chosen key-value store, we can solve this requirement by writing every message to disk (or batches of messages to disk), and once we have verified the message has been successfully committed to disk, respond to the original publisher with an acknowledgment.

```go
db, err := badger.Open(badger.DefaultOptions("/data/badger"))
// ...

nc.Subscribe("page_views", func(m *nats.Msg) {
    err := db.Update(func(txn *badger.Txn) error {
        msgKey := ksuid.New() // "naturally" sorted, globally unique identifier
        msgValue := m.Data
        err := txn.Set(msgKey, msgValue)
        return err
    })
    if err != nil {
        // ... Don't acknowledge.
    }
    // Message has been committed.
    m.Respond([]byte("Acknowledged"))
})
```

_Requirement #4 - We don't want the overhead of managing a large replicated service like Kafka._

By using Badger, we are able to mount an NFS volume on our container. This could, for example, run in [ECS](https://aws.amazon.com/ecs/) with [EFS](https://aws.amazon.com/efs/). The problem with ECS and EFS is that we don't have a [stable identifier to attach storage](https://github.com/aws/containers-roadmap/issues/127#issuecomment-656866073). All containers of ECS would utilize the same volume of EFS. This means we cannot simply tell our program to use the `/data/badger` directory. When Badger specifies a directory to `Open`, it acquires a lock on that directory. This means no other process may modify the directory while the instance of Badger is holding a lock. We can use this to our benefit by passing a unique id to the directory path for the Badger database. e.g., `/data/badger/instance-fe1ed453-c964-45dc-a5a6-ff6959b60dd9`. Now every instance of ECS will have it's own unique Badger database on our EFS volume.

The problem left is reaping zombie Badger databases when an ECS container shuts down. Badger was created specifically for the [Dgraph](https://github.com/dgraph-io/dgraph) database. One of the things Badger does exceptionally well is copy its contents by [streaming](https://github.com/dgraph-io/badger#stream) it out to another instance. Using this streaming process, we can have a background goroutine running on every instance of our service that periodically tries to acquire locks on the other Badger data directories. If the background goroutine is able to successfully acquire a lock on another directory, that means the ECS instance which created that Badger directory has shut down. At this point, we can stream the contents of that directory's database into the one actively being managed by this instance. By automatically reaping zombie directories, this will allow us to scale our ECS instances up and down automatically without having to do any kind of managed clustering.

```go
// loop over other instance directories.
// e.g.,
// /data/badger/instance-fe1ed453-c964-45dc-a5a6-ff6959b60dd9
// /data/badger/instance-392358a9-1d61-4a5c-86b9-1a8059f25dc8
// /data/badger/instance-89ce45ff-f925-4c50-9cec-677b06f475d9

// When we get a lock on one, stream the contents of that database into
// the database we created when the service started.
// 89ce45ff-f925-4c50-9cec-677b06f475d9 -> fe1ed453-c964-45dc-a5a6-ff6959b60dd9

// Finally, remove the database we just copied.
```

_Requirement #5 - We need acknowledgment capabilities between every producer, consumer, and webhook._

We've already touched on this requirement quite a bit. Any time a producer produces a message, that message must be acknowledge once it's been successfully processed between the end to end systems.

Let's circle back to Requirement #2. Now that we have our disk storage figured out by wiring up NATS to Badger, we have a way to guarantee messages are persisted to disk and will survive system restarts. When a message is destined for a webhook subject and that webhook does not respond in a reasonable amount of time, we can simply write it out to our dead letter queue. Let's see what that looks like.

```go
func sendToSaasWebhook1(msg *nats.Msg) error {
    err := webhook1HttpRequest(msg)
    if err != nil {
        // Publish the message to our DLQ subject.
        publishToDLQ(msg)
        return err
    }
    // Success writing to SaaS webhook.
    return nil
}
```

We'll design our dead letter queue so that it will automatically retry sending messages it receives. When it receives a message, the message will be persisted to Badger with a key that is unique and sortable. The key will internally contain a timestamp which will allow us to sort the keys by time. This means we can have a goroutine that runs in a loop looking for messages in Badger that are ready to be republished. Any timestamps earlier than the current time will signal that the message is ready to be republished back to the original subject. Doing this allows us to delay the republishing of messages by creating keys with embedded timestamps that occur in the future. If we wanted to try republishing a message 15 minutes from now, we would create a key with an embedded timestamp of 15 minutes from now. When the 15 minutes have passed, the current time will be greater than the key timestamp and the message will be picked up and republished.

![Diagram of a Retry Service](requeue.svg)

Instead of adding a Streaming technology between every producer and consumer, we have instead opted to allow producers to publish at any rate they wish. In the event the consumer is not able to keep up, the producer falls back to sending the message to our Retry Service, also known as requeue. Because our Retry Service is running in ECS it will automatically scale to the needs of our publishers. Since our Retry Service consumers are all part of the same queue group, NATS will load balance messages across the Retry Service instances. Upon receiving a message, the Retry Service will persist it to the Badger database on disk, using a key with a timestamp at some point in the future. When the time comes, that message will be picked up and republished back on the subject it was intended for. If the consumer, such as `webhook-1-consumer` still does not acknowledge the message, exponential backoff will come into play and the message will be retired again at some point in the future. This process may continue until `webhook1` comes back online and the message is successfully delivered to the webhook.

### Decoupling

You'll notice we're not making a connection to the Retry Service and webhook consumers directly. Instead of tightly coupling our producers and consumers by connecting them over something like gRPC, we're able to make connections to our NATS cluster and reuse those connections for our publishing and subscribing needs. This makes doing things like testing much simpler because we don't have to replicate our entire networking and infrastructure stack, instead, we can embed a NATS server in our tests. Additionally, refactoring and reuse of code become easier, allowing us to move publisher and subscriber code around as we see fit without the constraints of our network topology between services.

### Enhancements

**Backpressure**

Before I wrap things up, I want to talk about a few enhancements we can make to our service. For starters, the Retry Service allows us to handle backpressure automatically. When a producer starts to receive backpressure due to it sending messages faster than the consumers can acknowledge, it can fall back to the Retry Service which should autoscale. The Retry Service will then attempt to publish the messages again at some point in the future when the consumer is better able to handle the messages.

**Rate Limiting**

Rate limiting is something our consumers may have to deal with when writing to a webhook. Most SaaS APIs will specify a rate limit for each customer to ensure that a single customer does not overload the API for all customers. In the event that an API does reject messages due to a rate limit, it may send back an HTTP header specifying the amount of time the customer should wait before retrying. If this is the case, instead of a consumer acknowledging back to a producer that the message was successfully processed, it may instead opt to send back a message specifying that the message should be retried in the amount of time provided by the webhook response. The producer can, in this case, offload the message to our retry service, with a payload specifying how long the retry service should wait before trying to publish the message again.

**Time-To-Live and Expiration**

Sometimes for whatever reason, messages may end up in a permanent failure situation. Maybe the payload of the message is malformed and it will never succeed in posting to a webhook endpoint. In this case, we don’t want the message to sit around in our Badger database, retrying over and over until the end of time. What we can do here is specify a maximum amount of time the message should be kept in Badger. This value, known as a TTL, can be set when we commit the message to Badger. Periodically, Badger will look for messages with an expired TTL and delete them for us. We can take this one step further and upon publishing messages, allow producers to specify TTLs for messages in the message payload. This gives each producer the ability to determine how long a message should be kept in the Retry Service before being removed.

### Summary

Hopefully, you can see the power of this pattern. By offloading any messages that are not acknowledged to the auto scaling Retry Service, we ensure we don't lose messages. We are able to code our producers in a way that allows them to fall back onto the Retry Service, which will automatically take care of retrying the publishing messages to their original topic at a later time. With this pattern, any number of topics can be created with any number of consumers without adding any additional Streaming technologies between our producers and consumers. I’ve put together some code for what a retry service like this might look like. Feel free to [check it out on GitHub](https://github.com/nickpoorman/nats-requeue).

