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

    - PHP web application spawned by the FastCGI Process Manager, behind Nginx reverse proxy setup.

2. The web app is contained directly in a **web server**. In this case, the web server acts like an **application server**. Typical examples include:

    - PHP web apps running on Apache through _mod_php_. 

