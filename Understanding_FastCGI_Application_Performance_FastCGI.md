---
title: Understanding FastCGI </br > Application Performance FastCGI
description: Just how fast is FastCGI? How does the performance of a FastCGI application </br > compare with the performance of the same application implemented using a Web server API?
layout: page
---

[Mark R. Brown](https://www.linkedin.com/in/mark-brown-32a01b11/)

Open Market, Inc.  

10 June 1996  

##### Copyright © 1996 Open Market, Inc. 245 First Street, Cambridge, MA 02142 U.S.A.    

* * *

*   [1\. Introduction](#S1)
*   [2\. Performance Basics](#S2)
*   [3\. Caching](#S3)
*   [4\. Database Access](#S4)
*   [5\. A Performance Test](#S5)
    *   [5.1 Application Scenario](#S5.1)
    *   [5.2 Application Design](#S5.2)
    *   [5.3 Test Conditions](#S5.3)
    *   [5.4 Test Results and Discussion](#S5.4)
*   [6\. Multi-threaded APIs](#S6)
*   [7\. Conclusion](#S7)

* * *

### <a name="S1">1\. Introduction</a>

Just how fast is FastCGI? How does the performance of a FastCGI application compare with the performance of the same application implemented using a Web server API?

Of course, the answer is that it depends upon the application. A more complete answer is that FastCGI often wins by a significant margin, and seldom loses by very much.

Papers on computer system performance can be laden with complex graphs showing how this varies with that. Seldom do the graphs shed much light on _why_ one system is faster than another. Advertising copy is often even less informative. An ad from one large Web server vendor says that its server "executes web applications up to five times faster than all other servers," but the ad gives little clue where the number "five" came from.

This paper is meant to convey an understanding of the primary factors that influence the performance of Web server applications and to show that architectural differences between FastCGI and server APIs often give an "unfair" performance advantage to FastCGI applications. We run a test that shows a FastCGI application running three times faster than the corresponding Web server API application. Under different conditions this factor might be larger or smaller. We show you what you'd need to measure to figure that out for the situation you face, rather than just saying "we're three times faster" and moving on.

This paper makes no attempt to prove that FastCGI is better than Web server APIs for every application. Web server APIs enable lightweight protocol extensions, such as Open Market's SecureLink extension, to be added to Web servers, as well as allowing other forms of server customization. But APIs are not well matched to mainstream applications such as personalized content or access to corporate databases, because of API drawbacks including high complexity, low security, and limited scalability. FastCGI shines when used for the vast majority of Web applications.

### <a name="S2">2\. Performance Basics</a>

Since this paper is about performance we need to be clear on what "performance" is.

The standard way to measure performance in a request-response system like the Web is to measure peak request throughput subject to a response time constriaint. For instance, a Web server application might be capable of performing 20 requests per second while responding to 90% of the requests in less than 2 seconds.

Response time is a thorny thing to measure on the Web because client communications links to the Internet have widely varying bandwidth. If the client is slow to read the server's response, response time at both the client and the server will go up, and there's nothing the server can do about it. For the purposes of making repeatable measurements the client should have a high-bandwidth communications link to the server.

[Footnote: When designing a Web server application that will be accessed over slow (e.g. 14.4 or even 28.8 kilobit/second modem) channels, pay attention to the simultaneous connections bottleneck. Some servers are limited by design to only 100 or 200 simultaneous connections. If your application sends 50 kilobytes of data to a typical client that can read 2 kilobytes per second, then a request takes 25 seconds to complete. If your server is limited to 100 simultaneous connections, throughput is limited to just 4 requests per second.]

Response time is seldom an issue when load is light, but response times rise quickly as the system approaches a bottleneck on some limited resource. The three resources that typical systems run out of are network I/O, disk I/O, and processor time. If short response time is a goal, it is a good idea to stay at or below 50% load on each of these resources. For instance, if your disk subsystem is capable of delivering 200 I/Os per second, then try to run your application at 100 I/Os per second to avoid having the disk subsystem contribute to slow response times. Through careful management it is possible to succeed in running closer to the edge, but careful management is both difficult and expensive so few systems get it.

If a Web server application is local to the Web server machine, then its internal design has no impact on network I/O. Application design can have a big impact on usage of disk I/O and processor time.

### <a name="S3">3\. Caching</a>

It is a rare Web server application that doesn't run fast when all the information it needs is available in its memory. And if the application doesn't run fast under those conditions, the possible solutions are evident: Tune the processor-hungry parts of the application, install a faster processor, or change the application's functional specification so it doesn't need to do so much work.

The way to make information available in memory is by caching. A cache is an in-memory data structure that contains information that's been read from its permanent home on disk. When the application needs information, it consults the cache, and uses the information if it is there. Otherwise is reads the information from disk and places a copy in the cache. If the cache is full, the application discards some old information before adding the new. When the application needs to change cached information, it changes both the cache entry and the information on disk. That way, if the application crashes, no information is lost; the application just runs more slowly for awhile after restarting, because the cache doesn't improve performance when it is empty.

Caching can reduce both disk I/O and processor time, because reading information from disk uses more processor time than reading it from the cache. Because caching addresses both of the potential bottlenecks, it is the focal point of high-performance Web server application design. CGI applications couldn't perform in-memory caching, because they exited after processing just one request. Web server APIs promised to solve this problem. But how effective is the solution?

Today's most widely deployed Web server APIs are based on a pool-of-processes server model. The Web server consists of a parent process and a pool of child processes. Processes do not share memory. An incoming request is assigned to an idle child at random. The child runs the request to completion before accepting a new request. A typical server has 32 child processes, a large server has 100 or 200.

In-memory caching works very poorly in this server model because processes do not share memory and incoming requests are assigned to processes at random. For instance, to keep a frequently-used file available in memory the server must keep a file copy per child, which wastes memory. When the file is modified all the children need to be notified, which is complex (the APIs don't provide a way to do it).

FastCGI is designed to allow effective in-memory caching. Requests are routed from any child process to a FastCGI application server. The FastCGI application process maintains an in-memory cache.

In some cases a single FastCGI application server won't provide enough performance. FastCGI provides two solutions: session affinity and multi-threading.

With session affinity you run a pool of application processes and the Web server routes requests to individual processes based on any information contained in the request. For instance, the server can route according to the area of content that's been requested, or according to the user. The user might be identified by an application-specific session identifier, by the user ID contained in an Open Market Secure Link ticket, by the Basic Authentication user name, or whatever. Each process maintains its own cache, and session affinity ensures that each incoming request has access to the cache that will speed up processing the most.

With multi-threading you run an application process that is designed to handle several requests at the same time. The threads handling concurrent requests share process memory, so they all have access to the same cache. Multi-threaded programming is complex -- concurrency makes programs difficult to test and debug -- but with FastCGI you can write single threaded _or_ multithreaded applications.

### <a name="S4">4\. Database Access</a>

Many Web server applications perform database access. Existing databases contain a lot of valuable information; Web server applications allow companies to give wider access to the information.

Access to database management systems, even within a single machine, is via connection-oriented protocols. An application "logs in" to a database, creating a connection, then performs one or more accesses. Frequently, the cost of creating the database connection is several times the cost of accessing data over an established connection.

To a first approximation database connections are just another type of state to be cached in memory by an application, so the discussion of caching above applies to caching database connections.

But database connections are special in one respect: They are often the basis for database licensing. You pay the database vendor according to the number of concurrent connections the database system can sustain. A 100-connection license costs much more than a 5-connection license. It follows that caching a database connection per Web server child process is not just wasteful of system's hardware resources, it could break your software budget.

### <a name="S5">5\. A Performance Test</a>

We designed a test application to illustrate performance issues. The application represents a class of applications that deliver personalized content. The test application is quite a bit simpler than any real application would be, but still illustrates the main performance issues. We implemented the application using both FastCGI and a current Web server API, and measured the performance of each.

#### <a name="S5.1">5.1 Application Scenario</a>

The application is based on a user database and a set of content files. When a user requests a content file, the application performs substitutions in the file using information from the user database. The application then returns the modified content to the user.

Each request accomplishes the following:

1.  authentication check: The user id is used to retrieve and check the password.
2.  attribute retrieval: The user id is used to retrieve all of the user's attribute values.
3.  file retrieval and filtering: The request identifies a content file. This file is read and all occurrences of variable names are replaced with the user's corresponding attribute values. The modified HTML is returned to the user.  

Of course, it is fair game to perform caching to shortcut any of these steps.

Each user's database record (including password and attribute values) is approximately 100 bytes long. Each content file is 3,000 bytes long. Both database and content files are stored on disks attached to the server platform.

A typical user makes 10 file accesses with realistic think times (30-60 seconds) between accesses, then disappears for a long time.

#### <a name="S5.2">5.2 Application Design</a>

The FastCGI application maintains a cache of recently-accessed attribute values from the database. When the cache misses the application reads from the database. Because only a small number of FastCGI application processes are needed, each process opens a database connection on startup and keeps it open.

The FastCGI application is configured as multiple application processes. This is desirable in order to get concurrent application processing during database reads and file reads. Requests are routed to these application processes using FastCGI session affinity keyed on the user id. This way all a user's requests after the first hit in the application's cache.

The API application does not maintain a cache; the API application has no way to share the cache among its processes, so the cache hit rate would be too low to make caching pay. The API application opens and closes a database connection on every request; keeping database connections open between requests would result in an unrealistically large number of database connections open at the same time, and very low utilization of each connection.

#### <a name="S5.3">5.3 Test Conditions</a>

The test load is generated by 10 HTTP client processes. The processes represent disjoint sets of users. A process makes a request for a user, then a request for a different user, and so on until it is time for the first user to make another request.

For simplicity the 10 client processes run on the same machine as the Web server. This avoids the possibility that a network bottleneck will obscure the test results. The database system also runs on this machine, as specified in the application scenario.

Response time is not an issue under the test conditions. We just measure throughput.

The API Web server is in these tests is Netscape 1.1.

#### <a name="S5.4">5.4 Test Results and Discussion</a>

Here are the test results:

```
FastCGI  12.0 msec per request = 83 requests per second
API      36.6 msec per request = 27 requests per second
```

Given the big architectural advantage that the FastCGI application enjoys over the API application, it is not surprising that the FastCGI application runs a lot faster. To gain a deeper understanding of these results we measured two more conditions:

*   API with sustained database connections. If you could afford the extra licensing cost, how much faster would your API application run?

    ```
API      16.0 msec per request = 61 requests per second
    ```

    Answer: Still not as fast as the FastCGI application.
*   FastCGI with cache disabled. How much benefit does the FastCGI application get from its cache?

    ```
FastCGI  20.1 msec per request = 50 requests per second
    ```

    Answer: A very substantial benefit, even though the database access is quite simple.  

What these two extra experiments show is that if the API and FastCGI applications are implemented in exactly the same way -- caching database connections but not caching user profile data -- the API application is slightly faster. This is what you'd expect, since the FastCGI application has to pay the cost of inter-process communication not present in the API application.

In the real world the two applications would not be implemented in the same way. FastCGI's architectural advantage results in much higher performance -- a factor of 3 in this test. With a remote database or more expensive database access the factor would be higher. With more substantial processing of the content files the factor would be smaller.

### <a name="S6">6\. Multi-threaded APIs</a>

Web servers with a multi-threaded internal structure (and APIs to match) are now starting to become more common. These servers don't have all of the disadvantages described in Section 3\. Does this mean that FastCGI's performance advantages will disappear?

A superficial analysis says yes. An API-based application in a single-process, multi-threaded server can maintain caches and database connections the same way a FastCGI application can. The API-based application does not pay for inter-process communication, so the API-based application will be slightly faster than the FastCGI application.

A deeper analysis says no. Multi-threaded programming is complex, because concurrency makes programs much more difficult to test and debug. In the case of multi-threaded programming to Web server APIs, the normal problems with multi-threading are compounded by the lack of isolation between different applications and between the applications and the Web server. With FastCGI you can write programs in the familiar single-threaded style, get all the reliability and maintainability of process isolation, and still get very high performance. If you truly need multi-threading, you can write multi-threaded FastCGI and still isolate your multi-threaded application from other applications and from the server. In short, multi-threading makes Web server APIs unusable for practially all applications, reducing the choice to FastCGI versus CGI. The performance winner in that contest is obviously FastCGI.

### <a name="S7">7\. Conclusion</a>

Just how fast is FastCGI? The answer: very fast indeed. Not because it has some specially-greased path through the operating system, but because its design is well matched to the needs of most applications. We invite you to make FastCGI the fast, open foundation for your Web server applications.

* * *

**© 1995, Open Market, Inc. / [Mark R. Brown](https://www.linkedin.com/in/mark-brown-32a01b11/)**
