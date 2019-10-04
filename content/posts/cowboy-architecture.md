---
title: "Cowboy architecture and execution flow"
date: 2019-09-18T19:00:00+03:00
tags: ["erlang", "elixir"]
image: "/img/cowboy_generic_flow.png" 
---

Cowboy is *the* default HTTP server for Erlang/OTP. It's built on top of [Ranch](https://ninenines.eu/docs/en/ranch/2.0/guide/) which is a socket worker pool for TCP. Together they power most of the web apps written in Erlang or Elixir, including the ones built on [Plug](https://github.com/elixir-plug/plug) or [Phoenix](https://phoenixframework.org) framework.

<!--more--> 

> This article refers to Cowboy 2.6.3 and Ranch 1.7.1.

To start a cowboy http server:

``` erlang
cowboy:start_clear(my_http_listener, [{port, 8080}], #{env => #{dispatch => Dispatch}})
```

Under the hood, it initializes *ranch_listener_sup* supervision tree under *ranch_sup*. 
We'll look further into it by examining the execution flow of the HTTP request.

Moreover, if you're starting Plug under a supervision tree, you're actually starting *ranch_listener_sup* process too:

``` elixir
%{start: {:ranch_listener_sup, :start_link, _opts}} = 
  Plug.Cowboy.child_spec(scheme: :http, plug: MyPlug, options: [port: 8080])
```

## How cowboy handles HTTP request

The standalone Ranch app has a *ranch_sup* supervisor with *ranch_server* process used for storing internal configuration values in an ETS table (ETS is a built-in in-memory db).

The `cowboy:start_clear()` function starts a *ranch_listener_sup* supervision tree under *ranch_sup*.

The image below shows a diagram of the process tree and the related simplified code fragments:

![Cowboy generic flow](/img/cowboy_generic_flow.svg)

The *ranch_acceptor_sup* supervisor process requests to listen on a socket (i.e. port) from the given *Transport* (more on it later) and starts several *ranch_acceptor* children processes (10 by default) that await in the blocking `Transport:accept(LSocket)` calls for incoming client connections. The moment any of the acceptors receives a new socket connection, it sends *start_protocol* message to the *ranch_conns_sup* process.

When *ranch_conns_sup* receives the message it calls `Protocol:start_link(Socket)` to start a new connection process.

### Transport and Protocol modules

Ranch relies on 2 configurable modules: 

**Transport** is a module that implements *ranch_transport* behavior. It defines an interface to interact with a socket. Transports can be used for connecting, listening and accepting connections, but also for receiving and sending data. By default, ranch includes the following transports: *ranch_tcp* that wraps *gen_tcp* and *ranch_ssl* that wraps *ssl*.

**Protocol** is a module that implements *ranch_protocol* behavior. It starts a connection process and defines the protocol logic executed in this process. By default, cowboy includes *cowboy_clear* and *cowboy_tls* modules. However, both of them rely on *cowboy_http* or *cowboy_http2* modules internally.

In short, *Transport* knows how to interact with the socket and *Protocol* knows how to talk HTTP with the client.

### Connection process

After *ranch_conns_sup* starts a new connection process and hands the socket ownership to it, *cowboy_clear* delegates to the *cowboy_http* module to start the receive loop. Here's where the magic happens! 

It instructs the socket to send the first socket message to the connection process *as a process message* (`Transport:setopts(Socket, [{active, once}])`).

The *cowboy_http:loop()* receives the message, parses the request and starts a separate *request process* with `cowboy_stream:init(Req)`.

### Request process

The *cowboy_stream* module delegates to *cowboy_stream_h* by default which starts a new request process.

The job of the request process is to iterate over *Middlewares* and execute them in sequence. The default middlewares are *cowboy_router* that finds the route handler for the request url, and *cowboy_handler* that calls the selected handler.

After going through all of the middlewares, the request process terminates.

## How cowboy sends a response

The most curious thing about Cowboy 2 architecture is the fact that the request process isn't responsible for sending the response to the client. It's the job of the connection process.

The image below shows a cropped diagram of the execution flow for sending the response.

![Cowboy generic flow](/img/cowboy_reply.svg)

To send the response, you need to call `cowboy_req:reply(Status, Headers, Body, Req)` from the cowboy handler. It'll wrap the arguments into a process message and send it to the connection process (started by *cowboy_clear* protocol).

Meanwhile, connection process is waiting in the receive loop for such commands. When it receives the `{response, Status, Headers, Body}` message, it uses `cow_http:response` function from *cowlib* library to construct an HTTP response without the body, and calls `Transport:send` to send the whole response to the client socket.

> It's important to note here that by the time when the connection process sends that response, the request process could already be done and terminated.

----

This sums up behind the scenes action in Cowboy for a simple request and response cycle. I simplified and skipped some of the things, especially related to request parsing, but overall cowboy and ranch written in a cohesive and clear way and I would encourage people to have a look at it too.

## Sources

- [Cowboy & Ranch user guides](https://ninenines.eu/docs/)
- [Medium: Erlang/OTP architectures: cowboy](https://medium.com/@kansi/erlang-otp-architectures-cowboy-7e5e011a7c4f)
