# Guidelines for building and maintaining resilient services

## Clear scope

Define what the service should do. What is the breadth of functionality it should implement? It's rarely the case but it's also possible to overpartition your functionalities, which might lead to terrible hacks in the future, so be aware.  

Don't bloat your services. A recommended approach is to minimize a service to the amount of code that could be re-implemented in a two-week period.

## Present your API

Use a well know format to present your API. OpenAPI / Swagger is the standard nowadays, so there's no reason not to follow it.

## Proper versioning

Always version your api's when you do major changes. There's nothing worse than changing a contract without giving time to the consumer to adjust his implementation. The easiest thing is to put a version prefix inside the request route. For example:

- /api/v1/user/{id}
- /api/v2/user?userId={userId}
- /api/v3/getUser?id={id}

## Monitoring

### Metrics

If you can't see it work, then you don't know if it works. It's critical that you collect and monitor as many metrics as possible (and useful). You have no idea how much better you can understand your system when you setup a good monitoring solution.

A list of metrics that you should collect and monitor are:

- Errors count - this includes: invalid request data (client errors); internal server errors; network errors; unavailable external services;
- Total service throughput (rpm) / Endpoint throughput (rpm) - this gives you the opportunity to monitor the system load, to identify hotspots and unusual traffic spikes.
- Response times (per endpoint) - Really useful to identify slow operations. If you need finer granularity you can also monitor the response time per external service call. This is really useful when you want to understand which part of an operation is the bottleneck.
- Service health - Great to identify if the service is alive and responsive.
- Memory utilization - Great to identify the existence of a memory leak.
- CPU utilization - Great to identify excessive computation and to reduce your cloud bill.
- Request/Response size - Great to identify endpoints that are not limited by the amount of items they can return. I encountered a production case where we had an endpoint that was able to dump the items from an entire database collection for the last 24 hours (nearly 50k+ items) in a single request.
- Active requests count - Although the response times are giving a similar information, it can be a good indicator to confirm that there's slowness in the system.

Some of the most commonly used time series databases are:

- InfluxDB (Push-based)
- Prometheus (Pull-based)
- Graphite (Push-based)
- Amazon Cloudwatch
- Datadog

Some of them provide rich querying capabilities, but they differ in the way they aggregate and compute data. You should always do a bit of research to understand what might work best for you.

The most commonly used open-source metrics monitoring UI is Grafana. It has built-in connectors for the time-series databases mentioned above and it also provides rich dashboards with flexible configurations and the option to setup Alerts on top of the metrics that are visualized. It also has built-in notification channels for PagerDuty, Slack, Alertmanager, etc.

### Centralized logging

The absolute minimum is to log all exceptions/errors that occur in your system. Believe me, you don't want to SSH to every server to look for log files when an incident happens. It's way more convenient to open, for example, Kibana, click 2 buttons and see all logs for a given service in a nicely formatted, sorted way. So use a *centralized logging solution*! 

Some options are:

- ELK stack (Elasticsearch, Logstash, Kibana)
- Loggly
- Splunk

It's also good to log important operations like transfering money, updating deal statuses, and the reason for those actions (the event that triggered them). But this goes into the domain of system auditing, so you should be careful not to leak sensitive information to third-party providers or to blow your system with too much logs.  

Also, watch out who might have access to your logs. If you start logging data in plaintext or in an easily decodable format, a malicious employee can use this information as a weapon against your customers/company.

### Alerts and Incident notifications

Configure alerts that will notify you when a metric threshold is breached.  
Some options are:

- Grafana
- Prometheus Alertmanager
- Kibana
- Cloudwatch

Use the tool that you know you will check multiple times throughout the day. The last thing you want is a brilliant monitoring solution, that no one keeps an eye on. In our case, we use Slack for our day-to-day communication, so we configured Slack channel notifications and there's no way to miss when an Alert is triggered because at least one of the developers will be active and will see the "Alert channel" section blinking.  

There are multiple options to consider when we talk about alert notification channels, but I will list only few:

- Slack
- PagerDuty
- Email
- Webhooks
- Microsoft Teams
- Telegram

## Resilience policies

Transient faults happen. The network is unreliable. The external dependencies are unreliable. If you want a robust system, you need to introduce resilience policies in your application, wherever it makes sense.  
Those policies can be:

- Timeout policies - That's when beyond a certain wait, a success result is unlikely. You don't want to wait 5 minutes for a database to respond.
- Retry policies with some backoff strategy - Many faults are transient and may self-correct after a short delay. You don't want a network "hiccup" to interrupt your request.
- Circuit breakers - When a system is seriously struggling, failing fast is better than making users/callers wait. Protecting a faulting system from overload can help it recover.
- Fallback policies - Things will still fail occasionally - plan what you will do when that happens.

## Caching

Calls over the network are nearly *seven orders of magnitude slower* than in-memory function calls.  

Reduce roundtrips to external dependencies as much as possible. Also, querying the database every time is nonessential when the nature of your data is static, doesn't change very often or when your application tolerates eventual consistency. For example displaying the number of views for a video is not something that is critical to the user. We don't care if the user sees 500 views or 513 views. On the other side, working with money requires 100% consistency so you should not cache the amount of a user's bank account.  

### In-memory cache vs Distributed cache

In-memory is as fast as you can get, but it doesn't scale good and it prevents you from sharing cached data between services. Also if you have more than one instance of a given microservice, you will start experiencing problems if you need to sync the cache between those instances.  

On the other hand, distributed caches like *Reddis, Aerospike, Memcached, DynamoDB, Ehcache, etc*, allows you to scale-out and share data between services. But they come with the cost of increased latency, since your calls go through the wire. Plus, they introduce another level of fragility, because network failures can and will happen.

There are multiple strategies to keep a cache up-to-date:  

- In-place updates and rolling aggregations - Let's take "Views count" for example. Instead of computing the value everytime, just increment a counter on every Add/Remove action in your application.
- Listen to the operational log of the database - This approach is pretty solid because you just can't skip an operation when you listen to the op log. Imagine you have a SQL database and an "OnInsert" trigger that updates some piece of data. It's very easy to skip such updates on the application side when you forget or simply don't know about the existence of this trigger, or when you can't replicate the logic outside the database.
- Schedule a function that queries specific parts of the database and have it update the cache periodically. This approach brings the benefit of not putting constant pressure on the cache like the first 2 options, at the cost of reduced consistency.
- Listen to a change stream - Publish the change events to a messaging system, and have a worker that consumes those events and updates the cache. This approach brings the benefit of decoupling business logic from the operational details related to keeping the cache fresh.  

Keep in mind that some of those strategies might lead to data inconsistency. Take for example the "Change stream" strategy. Imagine you execute a user request, persist the new state of the data to a SQL database and the next step is to publish the "Change event" to a message bus. Between the "persist to SQL" and "publish to message bus" steps, you have a short window of time, where your application might crash, the network might fail, and you will skip propagating this change to the cache. In such cases, you will need an alternative solution that periodically refreshes the entire cache, just to make sure your systems does not play with stale data. (Tough life, right.)

There are a couple of things you need to consider before you start implementing a cache:  

- What to cache?
- When to cache?
- Where to cache?
- How long to cache? What expiration policies do I need in order to keep the cache small, fast and as efficient as possible?
- How to keep the cache up-to-date?
- How many services will consume the cache?
- What is the expected load on the cache?

## Traffic management

Request rate limiting.

Auto-scaling and load balancing based on thresholds or a schedule.  

Requests buffering - you can use a messaging system like RabbitMQ, Kafka, Amazon SQS, Amazon Kinesis, etc, to buffer all incoming requests until they are ready to be processed. This works great because you have a natural limitation on the amount of concurrent requests that will be processed by your services and prevents your systems from "clogging" when there are traffic spikes. This approach is not easily usable everywhere, but it works great with systems that are designed with "reactive" mindset (from the UI to the Backend).

## Security

Internet-facing services should always communicate in a secure way. For example, always enforce HTTPS over HTTP. Especially when you work with sensitive data. Keep in mind that your consumers always have *at least one "Man-in-the-middle"* that listens to their traffic and that is their Internet Service Provider (ISP).  

For backend services it's best if every services has a role-based access control. The "least-access-possible" rule should be applied. AWS IAM is a good example where you have a fine-grained control over the access policies. For example, the "Taxation" service should not have access to the "Reporting" service, if it's not designed to interact with it.