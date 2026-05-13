+++
author = "Emily Miller"
title = "A brief dive into CGI"
date = "2100-10-31"
description = "Common Gateway Interface (CGI) is one of the oldest technologies still in use today to make web applications."
categories = [ "misc" ]
+++

[RFC 3875](https://www.rfc-editor.org/rfc/rfc3875) is the first RFC to discuss CGI in depth. Notably, this is not a formal specification, but rather a description of how CGI is commonly done in practice. To give a brief introduction to the technology from this RFC:

>    The Common Gateway Interface (CGI) [22] allows an HTTP [1], [4]
>   server and a CGI script to share responsibility for responding to
>   client requests.  The client request comprises a Uniform Resource
>   Identifier (URI) [11], a request method and various ancillary
>   information about the request provided by the transport protocol.
>
>   The CGI defines the abstract parameters, known as meta-variables,
>   which describe a client's request.  Together with a concrete
>   programmer interface this specifies a platform-independent interface
>   between the script and the HTTP server.
>
>   The server is responsible for managing connection, data transfer,
>   transport and network issues related to the client request, whereas
>   the CGI script handles the application issues, such as data access
>   and document processing.

Ok so, to briefly summarize what this is saying:
 - A CGI server handles parsing of HTTP requests and the networking side of things
 - It then uses a CGI script to handle the application side of the interface

This fundamentally makes sense. This is a very similar to other technologies of the sort that you'll see, for instance PHP servers.

In general, the way this looks in practice is something like the following:
 - Spin up a single "CGI Server" process, which listens for packets on a specific port
 - When it recieves a new TCP connection, spin up a new thread to handle the connection and continue listening
    - This new thread will then parse the HTTP request headers
    - It will then pick a script to run based on the URI (this is implementation defined)
    - It will then run a script in an implementation defined manner 
    - And package its output into a new HTTP request

## Picking a Script

By default, this is usually done by indexing against a root directory. For instance, `/www/cgi-bin/`, may have all the CGI scripts for an application, and then the server will attempt to match the URI to a file within this directory. 

## Running the Script

When running the script, by default it will store HTTP headers in environment variables such as `QUERY_STRING`, `CONTENT_LENGTH`, `REMOTE_ADDR`, etc. In the RFC description of the protocol, it actually abstracts this away using the term "Meta Variable", and states this is again implementation defined. 

It will then pass the request body in through `stdin` and output of `stdout` will be used to send the reply.
