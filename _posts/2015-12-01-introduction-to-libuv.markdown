---
layout: post
title:  "libuv tutorial: Hello, World!"
date:   2015-12-01 18:52:50 +0100
categories: libuv tutorial
---

[libuv]: http://libuv.org/
[node.js]: https://nodejs.org
[c-wikipedia]: https://en.wikipedia.org/wiki/C_(programming_language)
[libev]: http://software.schmorp.de/pkg/libev.html
[libevent]: http://libevent.org/
[libuv-an-introduction]: https://nikhilm.github.io/uvbook/introduction.html
[libuv-docs]: http://docs.libuv.org/
[libuv-tutorial-gh]: https://github.com/eivindbergem/libuv-tutorial
[libuv-build-instructions]: https://github.com/libuv/libuv#build-instructions
[hello-server]: https://github.com/eivindbergem/libuv-tutorial/blob/master/hello-world/server.c

# Introduction
[libuv][libuv] is the event library at the core of [Node.js][node.js]. It is written in [C][c-wikipedia] and provides a cross-platform abstraction on top of Epoll in Linux, kqueue in BSD, and IOCP in Windows. In addition in comes with a thread pool and a lot of built in functions to provide asynchronous filesystem operations, all using the same basic interface. For anyone who has ever tried to tackle asynchronous IO using select(), libuv is the answer to your prayers. libuv was initially an abstraction on top of [libev][libev] – which in turn is a bloat-free version of [libevent][libevent] – in order for Node.js to work on windows, as libev was unix only.

The learning curve for getting into libuv can be quite steep. There is a quite comprehensive tutorial [here][libuv-an-introduction], but I found it to leave out a lot of the details making it a bit hard to follow. There is also a lot of information in the [official documentation][libuv-docs], but it requires that you have at least basic knowledge of libuv and works better as a reference manual.

This tutorial describes a very simple server that accepts connections and sends "Hello, World!" to each client before closing the connection.

# Code

The code for this program can be found on [github][libuv-tutorial-gh]. In order to compile the program you need to have libuv installed. If it's not already installed on your system, you can find [instructions on github][libuv-build-instructions].

To compile the hello world server, run:

{% highlight sh %}
$ make hello-server
{% endhighlight %}

You can test it by running:
{% highlight sh %}
$ ./hello-server
{% endhighlight %}

And then in another terminal, run:
{% highlight sh %}
$ nc localhost 1234
{% endhighlight %}

This should print "Hello, World!" in your terminal.

#### [hello-world/server.c][hello-server]:
{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <uv.h>

#define PORT 1234
#define BACKLOG 10

// Close client
void close_client(uv_handle_t *handle) {
    // Free client handle after connection has been closed
    free(handle);
}

// Write callback
void on_write(uv_write_t *req, int status) {
    // Check status
    if (status < 0) {
        fprintf(stderr, "Write failed: %s\n", uv_strerror(status));
    }

    // Close client handle
    uv_close((uv_handle_t*)req->handle, close_client);

    // Free request handle
    free(req);
}

// Callback for new connections
void new_connection(uv_stream_t *server, int status) {
    // Check status code, anything under 0 means an error.
    if (status < 0) {
        fprintf(stderr, "New connection error: %s\n", uv_strerror(status));
        return;
    }

    // Create handle for client
    uv_tcp_t *client = malloc(sizeof(*client));
    memset(client, 0, sizeof(*client));
    uv_tcp_init(server->loop, client);

    // Accept new connection
    if (uv_accept(server, (uv_stream_t*) client) == 0) {
        // Create write request handle
        uv_write_t *req = malloc(sizeof(*req));
        memset(req, 0, sizeof(*req));

        // Add a buffer with hello world
        char *s = "Hello, World!\n";
        uv_buf_t bufs[] = {uv_buf_init(s, (unsigned int)strlen(s))};

        // Write and call on_write callback when finished
        uv_write((uv_write_t*)req, (uv_stream_t*)client, bufs, 1, on_write);
    }
    else {
        // Accept failed, closing client handle
        uv_close((uv_handle_t*)client, close_client);
    }

}

int main(void) {
    // Initialize the loop
    uv_loop_t *loop;
    loop = malloc(sizeof(uv_loop_t));
    uv_loop_init(loop);

    // Create sockaddr struct
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));

    // Convert ipv4 address and port into sockaddr struct
    uv_ip4_addr("0.0.0.0", PORT, &addr);

    // Set up tcp handle
    uv_tcp_t server;
    uv_tcp_init(loop, &server);

    // Bind to socket
    uv_tcp_bind(&server, (const struct sockaddr*)&addr, 0);

    // Listen on socket, run new_connection() on every new connection
    int ret = uv_listen((uv_stream_t*) &server, BACKLOG, new_connection);
    if (ret) {
        fprintf(stderr, "Listen error: %s\n", uv_strerror(ret));
        return 1;
    }

    // Start the loop
    uv_run(loop, UV_RUN_DEFAULT);

    // Close loop and shutdown
    uv_loop_close(loop);
    free(loop);
    return 0;
}
{% endhighlight %}

The code for the hello world server might look complicated, but it is for the most part boiler plate. I'll go through the most important bits part by part.

#### main()
Libuv provides it's own functions in place of the normal unix socket functions. They work, for the most part, just like the normal ones, just that they work with handles and are cross-platform.

{% highlight c %}
// Set up tcp handle
uv_tcp_t server;
uv_tcp_init(loop, &server);
{% endhighlight %}

Unix uses file descriptors for working with IO, libuv uses handles. For a TCP stream, use the uv_tcp_t handle, which is an extension of the uv_stream_t handle. Libuv use a crude form of struct inheritance, so that if you cast a uv_tcp_t as a uv_stream_t, you can access it's members. You'll be doing this a lot when working with libuv, as you see from the code.

The TCP handle is tied to the loop using uv_tcp_init().

{% highlight c %}
// Listen on socket, run new_connection() on every new connection
int ret = uv_listen((uv_stream_t*) &server, BACKLOG, new_connection);
if (ret) {
    fprintf(stderr, "Listen error: %s\n", uv_strerror(ret));
    return 1;
}
{% endhighlight %}

To listen for incoming connections, uv_listen() is used with the server handle, casted as a uv_stream_t pointer, as uv_listen() works with all kinds of streams, not just TCP streams. It also takes a backlog argument, just as with the normal listen(), and a callback function pointer. listen() is not a blocking call, so it's not really the listen()-call that is asynchronous. The callback is called when the socket connected to the server handle is ready for reading, which means that there is a new connection.

{% highlight c %}
// Start the loop
uv_run(loop, UV_RUN_DEFAULT);
{% endhighlight %}

uv_listen() doesn't call the callback by itself, the callback is called by the event loop. So to get things running, start the loop. The second argument to uv_run() is the run mode. UV_RUN_DEFAULT means that the loop will run until there are no more active handles or requests.

#### new_connection()

{% highlight c %}
// Check status code, anything under 0 means an error.
if (status < 0) {
    fprintf(stderr, "New connection error: %s\n", uv_strerror(status));
    return;
}
{% endhighlight %}

The error handling in libuv is similar to the normal C error handling using errno, except that the error codes is put directly into the status code. If status is less than 0, there's an error, and status is equal to ```-errno```, that is, errno negated. ```uv_strerror()``` returns a message similar to the one returned by ```strerror()```.

{% highlight c %}
// Create handle for client
uv_tcp_t *client = malloc(sizeof(*client));
memset(client, 0, sizeof(*client));
uv_tcp_init(server->loop, client);
{% endhighlight %}

Each client connection needs a separate handle. The loop can be accessed through the server handle.

{% highlight c %}
// Create write request handle
uv_write_t *req = malloc(sizeof(*req));
memset(req, 0, sizeof(*req));

// Add a buffer with hello world
char *s = "Hello, World!\n";
uv_buf_t bufs[] = {uv_buf_init(s, (unsigned int)strlen(s))};

// Write and call on_write callback when finished
uv_write((uv_write_t*)req, (uv_stream_t*)client, bufs, 1, on_write);
{% endhighlight %}

To make a write request, create a request handle. Ther's no need to initialize this, that is taken care of inside uv_write(). uv_write() also takes an array of uv_buf_t structs, which has two members: base, a char pointer to a buffer, and len, an unsigned int with the length of the buffer. Use uv_buf_init() to create a uv_buf_t from our string literal.

You might notice that bufs[] is declared on the stack. This might seem odd because bufs[] may not be reachable by the time the write request is executed. The reason this works is because bufs[] is copied into a heap by uv_write(). Not sure why it's done in this way, but I'm guessing it's for a good reason. This means that you don't have to bother with freeing bufs[] afterwards. However, you do have to free what buf->base points to, if heap allocated. But in this program, using a string literal, there's one less free to worry about. The freeing, of the request handle and the buf->base, if applicable, should be done in the write callback.
