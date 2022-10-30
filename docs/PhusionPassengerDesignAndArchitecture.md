# Phusion Passenger Design and Architecture

## Intro

### Web application models and the role of the application server

Here is how a typical web applications work from the viewpoint of someone who wants to connect a web application to a web server.

A typical, isolated, web application accepts an HTTP request from some I/O channel, processes it internally, and outputs an HTTP response, which is sent back to the client. This is done in a loop, until the application is commanded to exit. This does not necessarily mean that the web app speaks HTTP directly; it just means that the web app accepts some kind of representation of an HTTP request.


                        3. HTTP response
                            (output)
                               ^
                               |
                               |
1. HTTP request ------> Web application 
    (input)              2. Processing


Some web apps are directly accessible through the HTTP protocol, while others are not. It depends on the language and framework that the web app is built on. 
