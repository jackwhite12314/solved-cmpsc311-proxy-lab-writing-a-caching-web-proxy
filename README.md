Download Link: https://assignmentchef.com/product/solved-cmpsc311-proxy-lab-writing-a-caching-web-proxy
<br>
<h1>1      Introduction</h1>

A Web proxy is a program that acts as a middleman between a Web browser and an <em>end server</em>. Instead of contacting the end server directly to get a Web page, the browser contacts the proxy, which forwards the request on to the end server. When the end server replies to the proxy, the proxy sends the reply on to the browser.

Proxies are useful for many purposes. Sometimes proxies are used in firewalls, so that browsers behind a firewall can only contact a server beyond the firewall via the proxy. Proxies can also act as anonymizers: by stripping requests of all identifying information, a proxy can make the browser anonymous to Web servers. Proxies can even be used to cache web objects by storing local copies of objects from servers then responding to future requests by reading them out of its cache rather than by communicating again with remote servers.

In this lab, you will write a simple HTTP proxy that caches web objects. For the first part of the lab, you will set up the proxy to accept incoming connections, read and parse requests, forward requests to web servers, read the servers’ responses, and forward those responses to the corresponding clients. This first part will involve learning about basic HTTP operation and how to use sockets to write programs that communicate over network connections. In the second part, you will add caching to your proxy using a simple main memory cache of recently accessed web content.

2      Logistics

This is an individual project.

<h1>3       Handout instructions</h1>

Download proxylab-handout.tar file from Canvas. Copy the handout file to a protected directory on the Linux machine where you plan to do your work, and then issue the following command:

linux&gt; tar xvf proxylab-handout.tar

This will generate a handout directory called proxylab-handout. The README file describes the various files.

<h1>4        Part I: Implementing a sequential web proxy</h1>

The first step is implementing a basic sequential proxy that handles HTTP/1.0 GET requests. Other requests type, such as POST, are strictly optional.

When started, your proxy should listen for incoming connections on a port whose number will be specified on the command line. Once a connection is established, your proxy should read the entirety of the request from the client and parse the request. It should determine whether the client has sent a valid HTTP request; if so, it can then establish its own connection to the appropriate web server then request the object the client specified. Finally, your proxy should read the server’s response and forward it to the client.

4.1       HTTP/1.0 GET requests

When an end user enters a URL such as http://web.mit.edu/index.html into the address bar of a web browser, the browser will send an HTTP request to the proxy that begins with a line that might resemble the following:

GET http://web.mit.edu/index.html HTTP/1.1

In that case, the proxy should parse the request into at least the following fields: the hostname, web.mit.edu; and the path or query and everything following it, /index.html. That way, the proxy can determine that it should open a connection to web.mit.edu and send an HTTP request of its own starting with a line of the following form:

GET /index.html HTTP/1.0

Note that all lines in an HTTP request end with a carriage return, ‘r’, followed by a newline, ‘
’. Also important is that every HTTP request is terminated by an empty line: “r
”.

You should notice in the above example that the web browser’s request line ends with HTTP/1.1, while the proxy’s request line ends with HTTP/1.0. Modern web browsers will generate HTTP/1.1 requests, but your proxy should handle them and forward them as HTTP/1.0 requests.

It is important to consider that HTTP requests, even just the subset of HTTP/1.0 GET requests, can be incredibly complicated. The textbook describes certain details of HTTP transactions, but you should refer to RFC 1945 for the complete HTTP/1.0 specification. Ideally your HTTP request parser will be fully robust according to the relevant sections of RFC 1945, except for one detail: while the specification allows for multiline request fields, your proxy is not required to properly handle them. Of course, your proxy should never prematurely abort due to a malformed request.

4.2     Request headers

The important request headers for this lab are the Host, User-Agent, Connection, and Proxy-Connection headers:

<ul>

 <li>Always send a Host header. While this behavior is technically not sanctioned by the HTTP/1.0 specification, it is necessary to coax sensible responses out of certain Web servers, especially those that use virtual hosting.</li>

</ul>

The Host header describes the hostname of the end server. For example, to access http://web. mit.edu/index.html, your proxy would send the following header:

Host: web.mit.edu

It is possible that web browsers will attach their own Host headers to their HTTP requests. If that is the case, your proxy should use the same Host header as the browser.

<ul>

 <li>You <em>may </em>choose to always send the following User-Agent header:</li>

</ul>

User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3

The header is provided on two separate lines because it does not fit as a single line in the writeup, but your proxy should send the header as a single line.

The User-Agent header identifies the client (in terms of parameters such as the operating system and browser), and web servers often use the identifying information to manipulate the content they serve. Sending this particular User-Agent: string may improve, in content and diversity, the material that you get back during simple telnet-style testing.

<ul>

 <li>Always send the following Connection header:</li>

</ul>

Connection: close

<ul>

 <li>Always send the following Proxy-Connection header:</li>

</ul>

Proxy-Connection: close

The Connection and Proxy-Connection headers are used to specify whether a connection will be kept alive after the first request/response exchange is completed. It is perfectly acceptable (and suggested) to have your proxy open a new connection for each request. Specifying close as the value of these headers alerts web servers that your proxy intends to close connections after the first request/response exchange.

For your convenience, the values of the described User-Agent header is provided to you as a string constant in proxy.c.

Finally, if a browser sends any additional request headers as part of an HTTP request, your proxy should forward them unchanged.

4.3     Port numbers

There are two significant classes of port numbers for this lab: HTTP request ports and your proxy’s listening port.

The HTTP request port is an optional field in the URL of an HTTP request. That is, the URL may be of the form, http://cse-cmpsc311.cse.psu.edu:8080, in which case your proxy should connect to the host cse-cmpsc311.cse.psu.edu on port 8080 instead of the default HTTP port, which is port 80. Your proxy must properly function whether or not the port number is included in the URL.

The listening port is the port on which your proxy should listen for incoming connections. Your proxy should accept a command line argument specifying the listening port number for your proxy. For example, with the following command, your proxy should listen for connections on port 8081:

linux&gt; ./proxy 8081

You may select any non-privileged listening port (greater than 1,024 and less than 65,536) as long as it is not used by other processes. Since each proxy must use a unique listening port and many people will simultaneously be working on each machine, the script port-for-user.pl is provided to help you pick your own personal port number. Use it to generate port number based on your user ID:

linux&gt; ./port-for-user.pl droh droh: 45806

The port, <em>p</em>, returned by port-for-user.pl is always an even number. So if you need an additional port number, say for the Tiny server, you can safely use ports <em>p </em>and <em>p</em>+1.

Please don’t pick your own random port. If you do, you run the risk of interfering with another user.

<h1>5        Part II: Caching your Requests</h1>

For the second part of the lab, you will add a cache to your proxy that stores recently-used Web objects in memory. HTTP actually defines a fairly complex model by which web servers can give instructions as to how the objects they serve should be cached and clients can specify how caches should be used on their behalf. However, your proxy will adopt a simplified approach.

When your proxy receives a web object from a server, it should cache it in memory as it transmits the object to the client. If another client requests the same object from the same server, your proxy need not reconnect to the server; it can simply resend the cached object.

Obviously, if your proxy were to cache every object that is ever requested, it would require an unlimited amount of memory. Moreover, because some web objects are larger than others, it might be the case that one giant object will consume the entire cache, preventing other objects from being cached at all. To avoid those problems, your proxy should have both a maximum cache size and a maximum cache object size.

5.1     Maximum cache size

The entirety of your proxy’s cache should have the following maximum size:

MAX_CACHE_SIZE = 16 MB (16777216 Bytes)

When calculating the size of its cache, your proxy must only count bytes used to store the actual web objects; any extraneous bytes, including metadata, should be ignored.

5.2     Maximum object size

Your proxy should only cache web objects that do not exceed the following maximum size:

MAX_OBJECT_SIZE = 8 MB (8388608 Bytes)

For your convenience, both size limits are provided as macros in proxy.c.

The easiest way to implement a correct cache is to allocate a buffer for the active connection and accumulate data as it is received from the server. If the size of the buffer ever exceeds the maximum object size, the buffer can be discarded. If the entirety of the web server’s response is read before the maximum object size is exceeded, then the object can be cached. Using this scheme, the maximum amount of data your proxy will ever use for web objects is the following:

MAX_CACHE_SIZE + MAX_OBJECT_SIZE

5.3      Eviction policy

Your proxy’s cache should employ an eviction policy that is a least-recently-used (LRU) eviction policy for your sequential proxy server. Notice that both reading an object from the cache and writing it into the cache count as using the object.

<h1>6      Evaluation</h1>

This assignment will be graded out of a total of 55 points:

<ul>

 <li><em>BasicCorrectness</em>: 30 points for basic proxy operation</li>

 <li><em>Cache</em>: 25 points for a working cache</li>

</ul>

6.1     Autograding

Your handout materials include an autograder, called driver.sh, that your instructor will use to get preliminary scores for <em>BasicCorrectness</em>, and <em>Cache</em>. From the proxylab-handout directory:

linux&gt; ./driver.sh

You must run the driver on a Linux machine.

The autograder does only simple checks to confirm that your code is acting like a caching proxy. For the final grade, we will do additional manual testing to see how your proxy deals with real pages. Here is a list of some pages that still uses http protocol (as of Nov. 14th 2018) that you can use to test.

<ul>

 <li>http://web.mit.edu</li>

 <li>http://www.espn.com</li>

 <li>http://www.bbc.com</li>

 <li>http://cse-cmpsc311.cse.psu.edu:8080</li>

</ul>

6.2     Robustness

As always, you must deliver a program that is robust to errors and even malformed or malicious input. Servers are typically long-running processes, and web proxies are no exception. Think carefully about how long-running processes should react to different types of errors. For many kinds of errors, it is certainly inappropriate for your proxy to immediately exit.

Robustness implies other requirements as well, including invulnerability to error cases like segmentation faults and a lack of memory leaks and file descriptor leaks.

<h1>7      Testing and debugging</h1>

Besides the simple autograder, you will not have any sample inputs or a test program to test your implementation. You will have to come up with your own tests and perhaps even your own testing harness to help you debug your code and decide when you have a correct implementation. This is a valuable skill in the real world, where exact operating conditions are rarely known and reference solutions are often unavailable.

Fortunately there are many tools you can use to debug and test your proxy. Be sure to exercise all code paths and test a representative set of inputs, including base cases, typical cases, and edge cases.

7.1     Tiny web server

Your handout directory the source code for the CS:APP Tiny web server. While not as powerful as thttpd, the CS:APP Tiny web server will be easy for you to modify as you see fit. It’s also a reasonable starting point for your proxy code. And it’s the server that the driver code uses to fetch pages.

7.2 telnet

As described in your textbook (11.5.3), you can use telnet to open a connection to your proxy and send it HTTP requests.

7.3 curl

You can use curl to generate HTTP requests to any server, including your own proxy. It is an extremely useful debugging tool. For example, if your proxy and Tiny are both running on the local machine, Tiny is listening on port 8080, and proxy is listening on port 8081, then you can request a page from Tiny via your proxy using the following curl command:

$ curl -v –proxy localhost:8081 http://localhost:8080

<ul>

 <li>About to connect() to proxy localhost port 8081 (#0)</li>

 <li>Trying ::1… Connection refused</li>

 <li>Trying 127.0.0.1… connected</li>

 <li>Connected to localhost (127.0.0.1) port 8081 (#0)</li>

</ul>

&gt; GET http://localhost:8080/ HTTP/1.1

&gt; User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.27.1 zlib/1.2.3

&gt; Host: localhost:8080

&gt; Accept: */*

&gt; Proxy-Connection: Keep-Alive

&gt;

<ul>

 <li>HTTP 1.0, assume close after body</li>

</ul>

&lt; HTTP/1.0 200 OK

&lt; Server: Tiny Web Server

&lt; Connection: close

&lt; Content-length: 121

&lt; Content-type: text/html

&lt;

&lt;html&gt; &lt;head&gt;&lt;title&gt;test&lt;/title&gt;&lt;/head&gt;

&lt;body&gt;

&lt;img align=”middle” src=”godzilla.gif”&gt;

Dave O’Hallaron

&lt;/body&gt;

&lt;/html&gt;

* Closing connection #0

7.4 netcat

netcat, also known as nc, is a versatile network utility. You can use netcat just like telnet, to open connections to servers. Hence, imagining that your proxy were running on localhost using port 8081 you can do something like the following to manually test your proxy:

$ nc localhost 8081

GET http://cse-cmpsc311.cse.psu.edu:8080 HTTP/1.1

HTTP/1.0 200 OK

MIME-Version: 1.0

Content-Type: text/html

Content-Length: 40922 ….

In addition to being able to connect to Web servers, netcat can also operate as a server itself. With the following command, you can run netcat as a server listening on port 12345:

sh&gt; nc -l 12345

Once you have set up a netcat server, you can generate a request to a phony object on it through your proxy, and you will be able to inspect the exact request that your proxy sent to netcat.

7.5     Web browsers

Eventually you should test your proxy using the <em>mostrecentversion </em>of Mozilla Firefox. Visiting About Firefox will automatically update your browser to the most recent version.

To configure Firefox to work with a proxy, visit

Preferences&gt;Advanced&gt;Network&gt;Settings

It will be very exciting to see your proxy working through a real Web browser. Although the functionality of your proxy will be limited, you will notice that you are able to browse the vast majority of websites through your proxy.

An important caveat is that you must be very careful when testing caching using a Web browser. All modern Web browsers have caches of their own, which you should disable before attempting to test your proxy’s cache.

<h1>8       Handin instructions</h1>

The provided Makefile includes functionality to build your final handin file. Issue the following command from your working directory:

linux&gt; make handin

The output is the file ../proxylab-handin.tar, which you can then handin.

Please make sure that the handin.tar file you submitted really works. You should download your submitted version, unpack in a fresh directory, enter make and test the generated proxy program. This is the last project of the semester and you will not have a chance to resubmit if you provide us a wrong copy.

Submit thie proxylab-handin.tar file to Canvas.

<ul>

 <li>Chapters 10-11 of the textbook contains useful information on system-level I/O, network programming, HTTP protocols.</li>

 <li>RFC 1945 (http://www.ietf.org/rfc/rfc1945.txt) is the complete specification for the HTTP/1.0 protocol.</li>

</ul>

<h1>9      Hints</h1>

<ul>

 <li>As discussed in Section 10.11 of your textbook, using standard I/O functions for socket input and output is a problem. Instead, we recommend that you use the Robust I/O (RIO) package, which is provided in the csapp.c file in the handout directory.</li>

 <li>The error-handling functions provide in csapp.c are not appropriate for your proxy because once a server begins accepting connections, it is not supposed to terminate. You’ll need to modify them or write your own.</li>

 <li>You are free to modify the files in the handout directory any way you like. For example, for the sake of good modularity, you might implement your cache functions as a library in files called cache.c and cache.h. Of course, adding new files will require you to update the provided Makefile.</li>

 <li>As discussed in the Aside on page 964 of the CS:APP3e text, your proxy must ignore SIGPIPE signals and should deal gracefully with write operations that return EPIPE errors.</li>

 <li>Sometimes, calling read to receive bytes from a socket that has been prematurely closed will cause read to return -1 with errno set to ECONNRESET. Your proxy should not terminate due to this error either.</li>

 <li>Remember that not all content on the web is ASCII text. Much of the content on the web is binary data, such as images and video. Ensure that you account for binary data when selecting and using functions for network I/O.</li>

 <li>Forward all requests as HTTP/1.0 even if the original request was HTTP/1.1.</li>

</ul>

Good luck!