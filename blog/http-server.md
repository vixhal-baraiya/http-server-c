# Building an HTTP Server from Scratch in C

In this article, we'll build a fully working HTTP server from scratch using pure C. No frameworks, no external libraries, just raw sockets, manual HTTP parsing, and direct interaction with the operating system networking APIs.

By the end of this, you'll have a server that:
- Opens a socket and listens on a port
- Accepts incoming connections
- Reads and parses an HTTP request
- Sends back a proper HTTP response

Let's get into it...
---

## Prerequisites

You should know basic C (pointers, structs, strings). You don't need to be a C expert, but if you've never written a `malloc` before, spend 30 minutes on that first.

You also need a Linux or macOS machine. Windows works too but the socket API is slightly different. This tutorial uses the POSIX socket API.

---

## Step 1: Understand What HTTP Actually Is

Before writing any code, it helps to know what an HTTP request and response look like at the text level.

When your browser visits `http://localhost:8080/`, it sends something like this:

```
GET / HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0
Accept: text/html

```

And the server is supposed to send back something like:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 13

Hello, World!
```

That's it. It's just text over a TCP connection. Your job is to:
1. Listen for a TCP connection
2. Read that text
3. Write back a response in the same format

---

## Step 2: Set Up the Project

Create a folder and a single C file:

```bash
mkdir http-server
cd http-server
touch server.c
```

You'll compile with:

```bash
gcc -o server server.c
./server
```

Now open `server.c` and add the includes you'll need:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
```

Here's what each one does:

- `stdio.h` and `stdlib.h` are the usual suspects for I/O and memory
- `unistd.h` gives you `read()`, `write()`, and `close()`
- `sys/socket.h` and `netinet/in.h` are for socket functions and the address structs
- `arpa/inet.h` is for converting IP addresses

---

## Step 3: Create a Socket

A socket is basically a file descriptor that represents a network connection. You create one with the `socket()` function.

Add this to `server.c`:

```c
#define PORT 8080

int main() {
    int server_fd;
    struct sockaddr_in address;
    int opt = 1;

    // Step 1: Create socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    printf("Socket created successfully\n");
    return 0;
}
```

The arguments to `socket()`:
- `AF_INET` means we're using IPv4
- `SOCK_STREAM` means TCP (as opposed to UDP which would be `SOCK_DGRAM`)
- `0` lets the OS pick the right protocol automatically

If `socket()` fails, it returns `-1` and sets `errno`. The `perror()` call prints a human-readable error message.

---

## Step 4: Set Socket Options

This step is optional but you'll thank yourself later. Without it, if you stop and restart your server quickly, you'll get an "Address already in use" error.

Add this right after the socket creation:

```c
    // Step 2: Set socket options
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt)) == -1) {
        perror("setsockopt failed");
        exit(EXIT_FAILURE);
    }
```

`SO_REUSEADDR` lets you reuse the port immediately after the server stops. Without this, the OS holds onto the port for a couple of minutes after you kill the process, which is super annoying during development.

---

## Step 5: Bind to a Port

Creating a socket doesn't attach it to any address or port. You need to `bind()` it.

```c
    // Step 3: Configure address and bind
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) == -1) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    printf("Bound to port %d\n", PORT);
```

A few things to note here:

`INADDR_ANY` means "listen on all network interfaces". So if your machine has multiple IP addresses, it'll accept connections on all of them.

`htons(PORT)` converts the port number from host byte order to network byte order. Different CPUs store multi-byte numbers differently (big-endian vs little-endian), and network protocols always use big-endian. `htons` stands for "host to network short".

---

## Step 6: Start Listening

Now tell the OS you're ready to accept connections:

```c
    // Step 4: Listen
    if (listen(server_fd, 10) == -1) {
        perror("listen failed");
        exit(EXIT_FAILURE);
    }

    printf("Listening on port %d...\n", PORT);
```

The second argument to `listen()` is the backlog. This is the maximum number of pending connections that can queue up while your server is busy handling another one. `10` is fine for a simple server.

---

## Step 7: Accept Connections

This is where the server actually waits for a client. `accept()` blocks (pauses) until someone connects.

```c
    // Step 5: Accept and handle connections
    int addrlen = sizeof(address);
    int client_fd;

    while (1) {
        client_fd = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen);
        if (client_fd == -1) {
            perror("accept failed");
            continue;
        }

        printf("Connection accepted\n");

        // We'll handle the request here
        // ...

        close(client_fd);
    }
```

The `while(1)` loop keeps the server running forever. Each time a client connects, `accept()` returns a new file descriptor (`client_fd`) for that specific connection. The original `server_fd` stays open and keeps listening.

---

## Step 8: Read the HTTP Request

Now that you have a connection, read what the client sent:

Replace the `// We'll handle the request here` comment with:

```c
        char buffer[4096] = {0};

        int bytes_read = read(client_fd, buffer, sizeof(buffer) - 1);
        if (bytes_read < 0) {
            perror("read failed");
            close(client_fd);
            continue;
        }

        printf("Received request:\n%s\n", buffer);
```

`read()` reads up to `sizeof(buffer) - 1` bytes from the client into `buffer`. The `-1` is so there's always room for a null terminator at the end, making it a valid C string.

At this point, if you compiled and ran the server and opened `http://localhost:8080` in a browser, you'd see the raw HTTP request printed in the terminal. That's kind of cool.

---

## Step 9: Parse the Request (Basic)

You don't need a full HTTP parser for this. Just figure out the method and the path.

Add a simple parsing function above `main()`:

```c
void parse_request(char *request, char *method, char *path) {
    sscanf(request, "%s %s", method, path);
}
```

Then use it in the loop:

```c
        char method[16] = {0};
        char path[256] = {0};
        parse_request(buffer, method, path);

        printf("Method: %s, Path: %s\n", method, path);
```

`sscanf` reads formatted input from a string. `%s` reads a whitespace-delimited token, so it'll grab `GET` and `/` from `GET / HTTP/1.1`.

This is a very basic parser. It handles the common case but doesn't deal with edge cases. For a real server you'd want something more robust, but this is enough to get moving.

---

## Step 10: Send an HTTP Response

Now for the fun part. Build a response and send it back.

Add a function to send a response:

```c
void send_response(int client_fd, int status_code, char *status_text, char *content_type, char *body) {
    char response[4096];

    int body_len = strlen(body);

    int response_len = snprintf(
        response,
        sizeof(response),
        "HTTP/1.1 %d %s\r\n"
        "Content-Type: %s\r\n"
        "Content-Length: %d\r\n"
        "Connection: close\r\n"
        "\r\n"
        "%s",
        status_code,
        status_text,
        content_type,
        body_len,
        body
    );

    write(client_fd, response, response_len);
}
```

Notice the `\r\n` line endings. HTTP requires carriage return + newline (`\r\n`) not just `\n`. A blank line (`\r\n\r\n`) separates the headers from the body. This is easy to get wrong, so double-check it.

---

## Step 11: Route the Request

Now use the parsed method and path to decide what to send back. Add this after the parsing code:

```c
        if (strcmp(method, "GET") == 0) {
            if (strcmp(path, "/") == 0) {
                send_response(client_fd, 200, "OK", "text/html",
                    "<html><body><h1>Hello from my C server!</h1></body></html>");
            } else if (strcmp(path, "/about") == 0) {
                send_response(client_fd, 200, "OK", "text/html",
                    "<html><body><h1>About Page</h1><p>Built with raw C sockets.</p></body></html>");
            } else {
                send_response(client_fd, 404, "Not Found", "text/html",
                    "<html><body><h1>404 - Not Found</h1></body></html>");
            }
        } else {
            send_response(client_fd, 405, "Method Not Allowed", "text/plain",
                "Method not allowed");
        }
```

`strcmp` compares two strings and returns 0 if they're equal. So `strcmp(method, "GET") == 0` means "if method equals GET".

---

## Step 12: The Full Code

Here's everything put together:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080

void parse_request(char *request, char *method, char *path) {
    sscanf(request, "%s %s", method, path);
}

void send_response(int client_fd, int status_code, char *status_text, char *content_type, char *body) {
    char response[4096];
    int body_len = strlen(body);

    int response_len = snprintf(
        response,
        sizeof(response),
        "HTTP/1.1 %d %s\r\n"
        "Content-Type: %s\r\n"
        "Content-Length: %d\r\n"
        "Connection: close\r\n"
        "\r\n"
        "%s",
        status_code,
        status_text,
        content_type,
        body_len,
        body
    );

    write(client_fd, response, response_len);
}

int main() {
    int server_fd;
    struct sockaddr_in address;
    int opt = 1;

    // Create socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // Set socket options
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt)) == -1) {
        perror("setsockopt failed");
        exit(EXIT_FAILURE);
    }

    // Bind to port
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) == -1) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    // Listen
    if (listen(server_fd, 10) == -1) {
        perror("listen failed");
        exit(EXIT_FAILURE);
    }

    printf("Server running on http://localhost:%d\n", PORT);

    int addrlen = sizeof(address);
    int client_fd;

    while (1) {
        // Accept connection
        client_fd = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen);
        if (client_fd == -1) {
            perror("accept failed");
            continue;
        }

        // Read request
        char buffer[4096] = {0};
        int bytes_read = read(client_fd, buffer, sizeof(buffer) - 1);
        if (bytes_read < 0) {
            perror("read failed");
            close(client_fd);
            continue;
        }

        // Parse request
        char method[16] = {0};
        char path[256] = {0};
        parse_request(buffer, method, path);

        printf("[%s] %s\n", method, path);

        // Route and respond
        if (strcmp(method, "GET") == 0) {
            if (strcmp(path, "/") == 0) {
                send_response(client_fd, 200, "OK", "text/html",
                    "<html><body><h1>Hello from my C server!</h1></body></html>");
            } else if (strcmp(path, "/about") == 0) {
                send_response(client_fd, 200, "OK", "text/html",
                    "<html><body><h1>About Page</h1><p>Built with raw C sockets.</p></body></html>");
            } else {
                send_response(client_fd, 404, "Not Found", "text/html",
                    "<html><body><h1>404 - Not Found</h1></body></html>");
            }
        } else {
            send_response(client_fd, 405, "Method Not Allowed", "text/plain",
                "Method not allowed");
        }

        close(client_fd);
    }

    close(server_fd);
    return 0;
}
```

---

## Step 13: Compile and Test

Compile:

```bash
gcc -o server server.c
./server
```

You should see:

```
Server running on http://localhost:8080
```

Now open your browser and go to:

- `http://localhost:8080/` - You should see "Hello from my C server!"
- `http://localhost:8080/about` - Should show the about page
- `http://localhost:8080/anything-else` - Should return a 404

You can also test with `curl`:

```bash
curl -v http://localhost:8080/
```

The `-v` flag shows the full request and response headers, which is great for seeing what's actually going on.

---

## Common Errors You'll Run Into

**"Address already in use"**

The port is still taken from a previous run. Either wait a few seconds, or kill the old process:

```bash
lsof -i :8080
kill <PID>
```

Or just use `SO_REUSEADDR` which you already set up in Step 4.

**Browser shows "Connection refused"**

The server isn't running, or it crashed on startup. Check your terminal output for error messages.

**Browser keeps loading forever**

You probably forgot to send a response, or the response format is wrong (missing `\r\n` at the end of headers, for example).

**Segfault when parsing**

Check that your buffer is null-terminated. If `read()` fills the entire buffer, `sscanf` has no null terminator to stop at. The `-1` in `read(client_fd, buffer, sizeof(buffer) - 1)` prevents this.

---

## What to Do Next

This server only handles one request at a time. While it's processing one client, all other connections are stuck in the queue. Here are some things you can add:

**Handle multiple connections with `fork()`**

After `accept()`, fork a child process to handle the request while the parent goes back to waiting:

```c
pid_t pid = fork();
if (pid == 0) {
    // Child process: handle the request
    close(server_fd);  // Child doesn't need the listener
    // ... handle request ...
    close(client_fd);
    exit(0);
} else {
    // Parent process: go back to accepting
    close(client_fd);  // Parent doesn't need this connection
}
```

**Serve static files**

Instead of hardcoded HTML strings, read files from disk and send them:

```c
FILE *f = fopen("index.html", "r");
if (f != NULL) {
    fread(body, 1, sizeof(body), f);
    fclose(f);
    send_response(client_fd, 200, "OK", "text/html", body);
}
```

**Parse headers properly**

A real HTTP parser reads the headers line by line and stores them in a struct. This lets you check things like `Content-Length` for POST bodies or `Accept` to serve different content types.

**Add threading**

Instead of forking, use `pthreads` to spawn a thread per connection. Threads are lighter than processes and share memory, which makes some things easier.