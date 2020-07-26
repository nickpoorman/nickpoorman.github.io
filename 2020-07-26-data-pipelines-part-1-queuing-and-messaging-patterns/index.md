# Data Pipelines: Part 1 - Queuing and Messaging Patterns


Current Data Pipelines are built using various queueing and messaging patterns. For the purpose of this blog post, I'm going to break them down into three categories by borrowing terms from [ZeroMQ](http://zguide.zeromq.org/page:all) and others. The first two, Publish-Subscribe and Push-Pull are in-memory (volatile memory). The third, we'll call Streaming which is persisted on disk (non-volatile memory).

### Publish-Subscribe

Publish-Subscribe (pub-sub), is a messaging pattern where producers (publishers) send messages to consumers (subscribers). Pub-sub allows publishers and subscribers to communicate without being tightly coupled. A publisher may publish messages to any number of topics, and a subscriber may selectively decide from which of those topics they would like to receive messages. Topics (sometimes referred to as subjects), are a way of grouping messages together and provide a means for subscribers to receive only the messages they are interested in while ignoring everything else.

![Diagram of Publish-Subscribe](pub-sub.svg)

Often publishers do not send messages directly to subscribers but instead send the messages through a relay, known as a broker. The broker, upon receiving a message to a topic from a publisher, will forward the message to all subscribers of that topic. If no subscribers are listening when a message is published, the message simply evaporates. This also means publishers can send the messages at any rate they choose, and the subscribers must keep up.

![Diagram of Publish-Subscribe With Broker](pub-sub-broker.svg)

### Push-Pull

Push-Pull, on the other hand, is similar to publish-subscribe in that publishers "push" a message into a queue, and the subscribers must request to pull the messages out. In Go, an unbuffered channel is essentially push-pull with a queue size of zero. The benefit here is that subscribers can control the rate at which they would like to receive the messages by explicitly requesting to pull messages at their own pace. The downside here is that if the publishers are producing data faster than the subscribers (the pullers) are requesting to consume it, you may run out of memory or resort to spilling messages to disk (in many cases improperly spilling to a slow swap partition).

![Diagram of Push-Pull](push-pull.svg)

### Streaming

Streaming is a concept built on top of push-pull. I'm referring to it as Streaming for lack of a better term (please reach out if you have one). Streaming tries to solve the downside of pub-sub and push-pull running out of memory. When either the producer or consumer is faster than the other, Streaming will queue up the messages on disk. Technologies such as Kafka attempt to horizontally scale this concept by distributing around a log of messages on disk.

![Diagram of Streaming](streaming.svg)

### Kinesis, Kafka, Google PubSub, ZeroMQ, NSQ, Redis Streams, ...

Over the last few years, it seems like the number of messaging systems created could rival the number of new JavaScript frameworks. The messaging systems essentially solve the same problem but abstract away different parts of the system by providing slightly different guarantees.

The main takeaway here is that no matter which system you consider, at their core, they either choose to keep messages in-memory or on-disk, and they choose to give you control of the publisher rate or the subscriber rate. That's not to say there isn't a happy middle-ground. If you look at protocols such as [TCP sliding windows](https://en.wikipedia.org/wiki/Sliding_window_protocol), you'll see that the producer and consumer can negotiate to establish a rate in which to produce and consume messages. Looking closer at TCP, you'll also notice a concept of [acknowledging messages for reliability](https://tools.ietf.org/html/rfc793#section-1.5), referred to as an ACK. Acknowledgment can be used to provide feedback from the consumer to the producer that a message was received. They can also be used to tell the producer to slow down if, for some reason, the consumer can't keep up. UDP, on the other hand, simply emits packets from a source and doesn't care if they arrive at the destination or the rate at which they arrive at the destination. UDP is similar to publish-subscribe in this sense.

### Data Pipelines and Back Pressure

In the case of Data Pipelines, you often are not able to control the producer rate, but you can control the consumer rate. This is because your producers are often users. For example, think about a Data Pipeline that must consume messages from every user that visits your website. Each page visit by a user produces a message so that later you can sum the number of times a given page was visited. With the scale of the web today, it's possible a page could become viral, and users visiting the page would quickly generate hundreds of millions of messages to your Data Pipeline. You wouldn't want to control the backpressure by telling visitors to stop visiting your site, and you certainly wouldn't want to try drinking directly from a fire hydrant by putting your mouth up to it. So instead, you opt to choose a technology where you control the consumer rate.

This is where Streaming comes in, and technologies such as Kinesis, Kafka, Google PubSub, etc.. begin building mountains of marketing material. Now you purchase a wonderful streaming technology, your users (the fire hydrants) can send messages as quickly as they please. Your Streaming technology accepts the messages and persists them out to disk (most likely) until they are ready to be consumed by you. All is right in the world. Now it's up to you to build a consumer.

### Consuming

Your Streaming technology is up and running, and your users are firing messages at unbounded rates. You want to build a consumer that pulls messages from your Streaming technology and totals up the times each page was visited. That should be simple enough. You write a small program that in a loop, pulls a message from your Streaming technology, looks at what page the message belongs to, and increments a counter. Maybe in the middle of the night, the traffic for your viral page begins to subside. All the while working at its own pace, your consumer continued to pull every message from your Streaming technology. It's has caught up to the rate of the messages being produced. You sit down at your computer in the morning and gaze over the glorious number of page views you received.

```json
{
  "visitTimestamp": "2020-07-14T18:47:13.919Z",
  "page": "/blog/data-pipeline-queuing-and-messaging-patterns",
  "publishedTimestamp": "2020-07-14T08:00:00.000Z"
}
```

```go
var totalPageViews int64
for {
    msg := pullMessageFromStreamingTechnology()

    // Increment our page views
    totalPageViews++

    // Tell streaming technology we processed the message
    // so that it can remove it from disk.
    msg.Acknowledge()
}
```

Imagine some time has passed, and after a while, you notice that the time of day you post new content to a page seems to have some correlation to how viral the page becomes. So you decided to pay for [Saas analysis tools](https://segment.com/catalog/) to determine the best time of day to publish new content. In order for the SaaS analysis tool to do its job, you must forward the events you receive from your users to the SaaS analysis tool. That way, the SaaS analysis tool can correlate the time you publish your articles to the time of the visits, thus telling you when the best time to publish is. How should we do this? One way is to hook into the consumer totaling up the number of page views. In addition to totaling the page views, we'll make it responsible for forwarding the message off to the SaaS analysis tool's [webhook](https://keen.io/docs/streams/extended-functionality/webhooks-integration/) endpoint.

![Diagram of a SaaS Webhook](saas-webhook.svg)

### Webhook Downtime

Like every service on the Internet, the SaaS analysis tool will experience downtime, and the webhook your Streaming technology consumer is forwarding messages to will stop responding. Maybe the SaaS analysis tool will be deploying an API update, or there will be a [widespread internet outage](https://news.ycombinator.com/item?id=23897705), causing the webhook to be offline for a significant amount of time. In the meantime, your consumer will stop making any progress because it continues to receive an error when trying to forward the message to the SaaS analysis tool's webhook.

```go
var totalPageViews int64
for {
    msg := pullMessageFromStreamingTechnology()

    err := sendMessageToAnalysisSaaSWebhook(msg)
    if err {
        // SaaS analysis tool isn't accepting messges right now;
        // probably because they don't have a redundant streaming technology.
        //
        // Don't acknowledge the message and try again.
        // Our Streaming Technology will return the same message to us again
        // on the next iteration of the loop when we ask for it.
        continue
    }

    // Increment our page views
    totalPageViews++

    // Tell streaming technology we processed the message
    // so that it can remove it from disk.
    msg.Acknowledge()
}
```

Is there something we can do here so that our consumer may continue pulling messages from our Streaming technology? We would really like to be able to ignore the fact the SaaS analysis tool is offline and continue totaling our page views to see what the total is. However, we can't permanently skip over messages we processed while the SaaS analysis tool was offline. We would need the messages to be delivered to the SaaS analysis tool when it comes back online.

What are some things we could do here? One option is for us to spend more money and buy a second Streaming technology (or buy one that allows for multiple topics) and place it in between our existing Streaming technology consumer and the SaaS analysis tool webhook.

![Diagram of a SaaS Webook With Streaming](saas-webhook-streaming.svg)

When consumer-1 pulls messages off streaming-1, it will push the message to streaming-2 and then increment the total page views. Consumer-2 can now pull messages from streaming-2 at its own pace. This means when SaaS Webhook is offline, consumer-2 isn't making any progress, but that shouldn't matter because it doesn't stop us from totaling up our page views. Once SaaS Webhook is back online, consumer-2 will send all the messages that arrived at streaming-2 while SaaS Webhook was offline.

```go
// consumer-1
var totalPageViews int64
for {
    msg := pullMessageFromStreaming1()

    sendMessageToStreaming2(msg)

    // Increment our page views
    totalPageViews++

    // Tell streaming technology we processed the message
    // so that it can remove it from disk.
    msg.Acknowledge()
}
```

```go
// consumer-2
for {
    msg := pullMessageFromStreaming2()

    err := sendMessageToAnalysisSaaSWebhook(msg)
    if err {
        // SaaS analysis tool isn't accepting messges right now;
        // probably because they don't have a redundant streaming technology.
        //
        // Don't acknowledge the message and try again.
        // Our Streaming Technology will return the same message to us again
        // on the next iteration of the loop when we ask for it.
        continue
    }

    // Tell streaming technology we processed the message
    // so that it can remove it from disk.
    msg.Acknowledge()
}
```

Additionally, this solution would solve any rate limiting SaaS Webhook may have in place. Most SaaS APIs set a limit on the maximum number of requests a single customer can make to it in a given time period so that a single customer cannot overload the API. In the solution above, because consumer-2 is able to pull messages from streaming-2 at it's own pace, consumer-2 can respect the SaaS Webhook rate limit by setting the pace equal to that of the rate limit.

### Up Next

Hopefully, now you can see how combining various queuing and messaging patterns such as pub-sub, push-pull, and streaming allows you to build resilient data pipelines. Given the solution above for sending messages to a webhook we don't control, you might be thinking that adding another set of streaming and consumer components for every additional webhook could get rather unwieldy. [In the next post](/2020-07-26-data-pipelines-part-2-retries/), I'll be going over an alternative solution that tries to address the amount of duplicate streaming and consumer components that would be necessary.

