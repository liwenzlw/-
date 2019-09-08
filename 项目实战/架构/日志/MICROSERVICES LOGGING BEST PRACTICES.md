[MICROSERVICES LOGGING BEST PRACTICES](https://www.scalyr.com/blog/microservices-logging-best-practices)



[Microservice architecture](http://microservices.io/) is an application structure that fosters the use of a loosely coupled system to allow you to develop, test, deploy, and release services independently of each other.

These services are part of a unique system, and the idea behind using microservices is to break a big problem into smaller problems. Usually, each service interacts with the others through an HTTP endpoint, hiding the details of its technology stack by exposing only a contract to its consumers.

Service A will call Service B, which at the same time calls Service C. Once the request chain is complete, Service A might be able to respond to the end customer that initiated the request.

Microservice architecture offers a lot of great benefits like the ability to use different technology stacks, deploy independently, solve small problems one at a time, and more!

## Microservice Complexity: The Motivation for Enumerating these Best Practices

But using microservices comes with a high cost in that they are complex…not only in how they communicate but also in how to manage them. And they get even more complicated when one or more services fail.

- Which service failed?
- Why?
- And under what circumstances? 

All these questions are hard to answer if you don’t have good, meaningful logs.

And let’s be honest, we all hate those “unknown” or “something went wrong” system errors. I myself have struggled with the problems that come from a lousy logging strategy. **In this post, I’ll share a few best practices that have helped me when dealing with microservices.**



## 1. Correlate Requests With a Unique ID

Think back to the request calling chain between Services A, B, and C that I talked about in the previous section. Following from this idea, it’s a good practice to tag each call with a unique ID that identifies the request.

For example, let’s say you’re logging access and error logs for each service. If you find an error in Service B, it might be useful for you to know whether the error was caused by the request coming from Service A or the request sent to Service C.

Maybe the error is informative enough that you don’t even need to reproduce it. But if that isn’t the case, the correct way to reproduce the error is to know all possible requests in the services that are related to, say, Service B.

When you have a correlation request ID, then you only need to look for that ID in the logs. And you’ll get all logs from services that were part of the main request to the system.

You’ll also know which service the main request spends the most time in, if the service is using the cache, if a service is calling other services more than once, and a lot of other interesting details.

## 2. Include a Unique ID in the Response

There are bound to be times when the microservice’s users will be facing an error. Don’t miss out on the opportunity to find out what caused the error!

You should code the response the client receives so that it contains a unique ID along with any other useful information about the error. This unique ID could be the same one you used to correlate the requests, as I discussed above.

Having a unique ID in the response payload of the request will help you and your customer identify problems more quickly. You’ll know the parameters of the request—date, time, and other details—which will help you better understand the problem.

Supplement common error messages like “contact the service administrator to report the problem” with an ID that labels the request. That way you can learn what caused the error and prevent it from happening again in the future.

## 3. Send Logs to a Centralized Location

Let’s assume that you’re already adding all sorts of useful information to your logs. But it’s *essential* to send logs to a centralized location.

Think about it.

If you have to log in to each individual server to read logs, you’ll spend more time trying to correlate problems than if you just had one location where you could access all logs. Also, systems usually get more complicated as time goes by, so the amount of microservices usually grows too. And to make things even more complicated, services could be hosted on different servers or providers.

Centralized logging is becoming the norm, especially if you’re working with cloud, container, or hybrid environments because the servers could be terminated without any notice. For example, containers are terminated if there’s an unexpected error, or the memory reaches 100 percent of its consumption capacity.

You could solve this problem by having agents pulling and pushing logs every five minutes or before servers get terminated. You could also configure a cronjob in the server, a sidecar container, or a shared file location where another process could centralize logs.

Avoid the temptation of [building a solution stack by yourself](https://www.scalyr.com/blog/the-build-vs-buy-decision-tree/), as where to centralize logging is a well-known problem that’s already [solved](https://www.scalyr.com/blog/log-management-what-is-it-and-why-you-need-it/).

Having logs from all of your services in one place makes it easy and efficient to correlate problems.

## 4. Structure Your Log Data

It’s going to be almost impossible to have a defined format for log data; some logs might need more fields than others, and those that don’t need all those excess fields will be wasting bytes.

Microservice architecture addresses this issue by using different technology stacks, which impacts the log format of each service. One service might use a comma to separate fields, while others use pipes or spaces.

All of this can get pretty complicated. Make the process of parsing logs simpler by structuring your log data into a standard format like [JavaScript Object Notation](https://www.json.org/) (JSON). JSON allows you to have multiple levels for your data so that, when necessary, you can get more semantic info in a single log event.

Parsing is also more straightforward than dealing with particular log formats. With structured data, the format of logs is standard, even though logs could have different fields.

You’ll also be able to create searches in the centralized location like looking for logs that have an “http_code” of 500 and above. Use structured logging to have a standard but flexible format in your microservices logs.

## 5. Add Context to Every Request

I don’t know about you, but when something goes wrong, I want to know everything!

Having enough information about an issue provides you with important context for the request. Knowing what could have caused the problem is essential, and having the right context will help you to understand what’s happening more quickly.

But adding context to logs could become a repetitive task in code because there’s going to be common data like date and time that you’ll need in each log event. So in code, logging will look simpler because you’ll only be logging the message and other unique areas.

You might want to log all the data you can get. But let me give you some specific fields that could help you figure out what you *really* need to log.

- Date and time. It doesn’t have to be UTC as long as the timezone is the same for everyone that needs to look at the logs.
- Stack errors. You could pass the exception object as a parameter to the logging library.
- The service name or code, so that you can differentiate which logs are from which microservice.
- The function, class, or file name where the error occurred so that you don’t have to guess where the problem is.
- External service interaction names—you’ll know which call to the DB was the one with the problem.
- The IP address of the server and client request. This information will make it easy to spot an unhealthy server or identify a [DDoS attack](https://en.wikipedia.org/wiki/Denial-of-service_attack).
- User-agent of the application so that you know which browsers or users are having issues.
- HTTP code to get more semantics of the error. These codes will be useful to create alerts.

Contextualizing the request with logs will save you time when you need to troubleshoot problems in the system.

## 6. Write Logs to Local Storage

Writing logs to local storage might sound contradictory to what I said previously about sending logs to a centralized location.

But hear me out.

The first time I read about sending logs somewhere else, I decided to send logs directly via an HTTP request.

Not a bad idea, but not a great one either. There were times when I had too much outbound network traffic. I ended up affecting calls to other more critical microservices.



There are always going to be trade-offs when sending logs to local storage. But I prefer this option because it helps to decouple logging and reduce context switching within the application.

You might want to separate storage volumes for logs from the ones used for the application, especially if it’s a database. [Amazon Web Services](https://aws.amazon.com/) (AWS), for example, has the option of mounting a volume using a service called [Elastic File System](https://aws.amazon.com/efs/) (EFS), which acts like [network-attached storage](https://en.wikipedia.org/wiki/Network-attached_storage) (NAS). You could spin up another server mounting the same volume in it and then forward logs to a centralized location.

Try to keep things simple by sending all application logs to the same location—using Docker containers fosters this behavior. Move the responsibility of [aggregating](https://www.scalyr.com/blog/what-is-log-aggregation-and-how-does-it-help-you/), filtering, and forwarding logs to some other processes or services.

## **7. Add Traces Where It Matters**

[Distributed tracing](https://www.scalyr.com/blog/distributed-tracing-important-2019) is becoming a big topic, especially in the microservices world. Distributed tracing is about adding traces between each call all over the system or even within a service.

What does that even mean?

Well, every call in the system is recorded to know how much time it took to finish a code execution. You could have a trace to know how much time it took for a microservice to respond. Or you could have a trace for a database call in the service. How many traces you add will depend on how much detail you want or need.

When you add traces and the latency in the system increases, it’s easy to spot where the problem might be.

You don’t have to add traces manually all over your code. Instead, you could instrument your code by using libraries like[ OpenCensus](https://www.scalyr.com/blog/opencensus-what-it-is/).

Just include the library in your code, and you’ll get standard metrics and traces that I’ve talked about in this post. Once you add telemetry with OpenCensus, you can export that data to other places, like[ Zipkin](https://www.scalyr.com/blog/zipkin-tutorial-distributed-tracing/) or into a centralized storage location.

## 8. Log Useful and Meaningful Data to Avoid Regret

These are just some ordinary things that help you log microservices. If you’re just starting out with logging, these practices might not make much sense and seem useless.

But once you’ve been using microservices for a while, they’ll save you a lot of trouble. Just make sure you’re always evaluating what you’re logging.

You’ll have a better idea of what’s important to log when you’re troubleshooting and say to yourself “I wish I had X and Y information so I could spot these weird errors more easily.”

Once you’re logging enough data, it’s time to automate things like alerts. You shouldn’t have to spend a ton of time reading logs to spot problems. Automating alerts will also help you to be proactive and keep errors from spreading to all of your users.

Having a centralized logging location to make further analysis is a must in microservices. Adding enough context to logs will make the difference in identifying what log data is useful, and what data is useless.