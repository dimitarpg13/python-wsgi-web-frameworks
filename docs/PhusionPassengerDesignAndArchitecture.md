# Phusion Passenger Design and Architecture

## Intro

### Web application models and the role of the application server

Here is how a typical web applications work from the viewpoint of someone who wants to connect a web application to a web server.

A typical, isolated, web application accepts an HTTP request from some I/O channel, processes it internally, and outputs an HTTP response, which is sent back to the client. This is done in a loop, until the application is commanded to exit. This does not necessarily mean that the web app speaks HTTP directly; it just means that the web app accepts some kind of representation of an HTTP request.

```
                        3. HTTP response
                            (output)
                               ^
                               |
                               |
1. HTTP request ------> Web application 
    (input)              2. Processing
```

Some web apps are directly accessible through the HTTP protocol, while others are not. It depends on the language and framework that the web app is built on. It depends on the language and framework that the web application is built on. For example, Ruby (Rack/Rails) and Python (WSGI) web applications are typically not directly accessible through the HTTP protocol. The reasons for this are historical.

#### Common models

Here are some common models that are in use:

1. The web app is contained in an **application server**. This application server may or  may not be able to contain multiple web applications. The administrator then connects the application server to a web server through some kind of protocol. This protocol may be HTTP, FastCGI, SCGI, AJP, etc. The web server dispatches (forwards) requests to the application server, which in turn dispatches requests to the correct web application, in a format that the web application understands.
   Conversely, HTTP responses outputted by the web application are sent to the application server, which in turn sends them to the web server, and eventually to the HTTP client.
Typical examples of such a model:
    - A J2EE application, contained in the Tomcat application server, reverse proxied behind the Apache web server. Tomcat can contain multiple web applications in a single Tomcat instance.

    - Most Ruby application servers besides _Phusion Passenger_ - _Thin_, _Unicorn_, _Goliath_, etc. These application servers can only contain a single Ruby web application per instance. They load the web application into their own process and are put behind a web server (_Apache_, _Nginx_) in a reverse proxy setup.

    - _Green Unicorn_, the Python WSGI application server, behind a reverse proxy setup.

    - _PHP_ web application spawned by the _FastCGI Process Manager_, behind _Nginx_ reverse proxy setup.

2. The web app is contained directly in a **web server**. In this case, the web server acts like an **application server**. Typical examples include:

    - _PHP_ web apps running on Apache through _mod_php_. 
    - Python WSGI web applications running on Apache through _mod_uwsgi_ or _mod_python_.

    Note that this does not necessarily mean that the web application is run inside the same process as the web server: it just means that the web server manages applications. In case of _mod_php_, PHP runs directly inside the Apache worker processes, but in case of _mod_uwsgi_ the Python processes can be configurred to run out-of-process.

    _Phusion Passenger for Apache_ and _Phusion Passenger for Nginx_ implement this model, and run applications outside the web server process. 

3. The web application *is* a web server, and can accept HTTP requests directly. Examples of this model:

    - Almost all _Node.js_ and _Meteor JS_ web applications.
    - The _Trac_ bug tracking software, running in its standalone server.

    In most setups, the admin puts those in a reverse proxy configuration, behind a real web server such as _Apache_ or _Nginx_, instead of letting hem acept HTTP requests directly. 

    _Phusion Passenger Sandalone_ implements this model. However, you can expose _Phusion Passenger Standalone_ directly to the Internet because it uses _Nginx_ internally.

4. The web application does not speak HTTP directly, but it is connected directly to the web server through some communication adapter. _CGI_, _FastCGI_ and _SCGI_ are good examples of this.


The above models cover how nearly all web apps work, whether they are based on _PHP_, _Django_, _J2EE_, _ASP.NET_, _Ruby on Rails_, etc. Note that all of these models provide the same functionality i.e. no model can do something that a different model can't_ and _SCGI_ are good examples of this.


The above models cover how nearly all web apps work, whether they are based on _PHP_, _Django_, _J2EE_, _ASP.NET_, _Ruby on Rails_, etc. Note that all of these models provide the same functionality i.e. no model can do something that a different model can't. All of those models are identical to the one described in the first diagram, if the combination of web servers, app serves, web applications are considered to be a single entity - a black box.

It should also be noted that these models do not enforce any particular I/O processing implementation. The web servers, app serves, web apps could process IO serially (i.e. one request at a time), could multiplex IO with a single thread (using `select(2)` or `poll(2)`) or it could process IO with multiple threads and/or multiple processes. It depends on the implementation.

Of course, there are many variations possible. For example, load balancers could be used.


#### The rationale behind reverse proxying

Administrators often put the web application or its application werver behnind a real web server in a reverse proxy setup, even when the web app/app server already speaks HTTP. This is because implementing HTTP in a proper, secure way involves more than just speaking the protocol. The public Internet is a hostile environment where clients can send any arbitrary data and can exhibit any arbitrary IO patterns. If you don't properly implement IO handling, then you could open
yourself either to parser vulnerabilities or DoS attacks.

Web servers like Apache and Nginx have already implemented world-class IO and connection handling code and it would be waste not to use it. In the end, putting the application in a reverse proxying setup often makes the whole system more robust and more secure. This is the reason why it is considered a good practice.

A typical problem involves dealing with **slow clients**. These clients may send HTTP requests slowly and read HTTP responses slowly, perhaps taking many seconds to complete their work. A naive single-threaded HTTP server implementation that reads an HTTP requests, processes and sends the HTTP response in a loop may end up spending so much time waiting for IO that spends very little time doing actual work. Worse: suppose that the client is malicious, just leaves the socket open and never
reads the HTTP response, then the server will spend forever waiting for the client, not being able to handle any more requests.

##### An example of naive HTTP server implementation
```

```


