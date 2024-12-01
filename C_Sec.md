# Cybersecurity with C Language

Welcome to the **Cybersecurity with C Language** guide! This document explores the fundamentals of cybersecurity programming in C, including topics such as networking with sockets, web scraping, handling HTTP requests, and security best practices. If you’re interested in understanding how C is used in cybersecurity contexts, this guide is for you.

---

## Table of Contents

1. [Introduction to Cybersecurity with C](#introduction-to-cybersecurity-with-c)
   - [Why Use C for Cybersecurity?](#why-use-c-for-cybersecurity)
   - [Common Security Libraries](#common-security-libraries)
2. [Working with Sockets](#working-with-sockets)
   - [Creating a Simple TCP Server](#creating-a-simple-tcp-server)
   - [Creating a Simple TCP Client](#creating-a-simple-tcp-client)
3. [Handling HTTP Requests](#handling-http-requests)
   - [Making HTTP Requests with `libcurl`](#making-http-requests-with-libcurl)
   - [Parsing HTTP Responses](#parsing-http-responses)
4. [Web Scraping in C](#web-scraping-in-c)
   - [Basic Web Scraping with `libcurl` and `HTML Parsing`](#basic-web-scraping-with-libcurl-and-html-parsing)
5. [Security Practices in C](#security-practices-in-c)
   - [Buffer Overflow Protection](#buffer-overflow-protection)
   - [Cryptography with C](#cryptography-with-c)
6. [Conclusion](#conclusion)
7. [Resources](#resources)

---

## Introduction to Cybersecurity with C

### Why Use C for Cybersecurity?

C is a **low-level language** that allows fine-grained control over system resources, which is critical when dealing with **networking, memory management**, and **system vulnerabilities**. Many cybersecurity tools and exploits are written in C due to its performance, portability, and direct access to hardware.

In cybersecurity, C is often used for tasks such as:
- Writing **networking tools** like scanners, servers, and clients.
- Developing **exploit code** or understanding system vulnerabilities.
- Implementing **cryptographic algorithms**.
- Writing tools for **web scraping**, **request handling**, and **protocol analysis**.

### Common Security Libraries

Several libraries are commonly used in cybersecurity with C. These libraries provide powerful functionality for network communication, cryptography, and other security tasks.

1. **libcurl**: A versatile library for making HTTP requests, FTP, and other protocols.
2. **OpenSSL**: Provides cryptographic functions, such as encryption, hashing, and SSL/TLS support.
3. **Libpcap**: A library for packet capturing and analysis.
4. **Nmap**: A network scanning library used for network discovery and vulnerability scanning.
5. **Pcapy**: A Python wrapper for Libpcap, useful in network analysis (though it’s Python-based, it can be used with C-based projects).

---

## Working with Sockets

Networking and sockets are central to cybersecurity, especially in areas like **penetration testing**, **network scanning**, and **exploit development**. C allows you to work with sockets directly, which is essential for creating server-client architectures or analyzing network traffic.

### Creating a Simple TCP Server

In C, the **socket** API allows you to create network applications. Here's an example of a simple TCP server using **POSIX sockets**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define BACKLOG 10

void handle_client(int client_sock) {
    char buffer[256];
    int n;

    // Read message from client
    memset(buffer, 0, 256);
    n = read(client_sock, buffer, 255);
    if (n < 0) {
        perror("Error reading from socket");
        exit(1);
    }

    printf("Message from client: %s\n", buffer);

    // Send response to client
    n = write(client_sock, "Message received", 16);
    if (n < 0) {
        perror("Error writing to socket");
        exit(1);
    }

    close(client_sock);
}

int main() {
    int sockfd, client_sock;
    socklen_t clilen;
    struct sockaddr_in serv_addr, cli_addr;

    // Create socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("Error opening socket");
        exit(1);
    }

    // Clear server address structure
    memset((char *)&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = INADDR_ANY;
    serv_addr.sin_port = htons(PORT);

    // Bind socket
    if (bind(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        perror("Error binding");
        exit(1);
    }

    // Listen for incoming connections
    listen(sockfd, BACKLOG);
    printf("Server listening on port %d...\n", PORT);

    clilen = sizeof(cli_addr);
    while (1) {
        // Accept incoming connection
        client_sock = accept(sockfd, (struct sockaddr *)&cli_addr, &clilen);
        if (client_sock < 0) {
            perror("Error accepting connection");
            continue;
        }

        // Handle the client
        handle_client(client_sock);
    }

    close(sockfd);
    return 0;
}
```

### Creating a Simple TCP Client

The following is an example of a simple TCP client that connects to a server and sends a message:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define SERVER_ADDR "127.0.0.1"
#define PORT 8080

int main() {
    int sockfd;
    struct sockaddr_in server_addr;
    char message[] = "Hello from client!";

    // Create socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("Error opening socket");
        exit(1);
    }

    // Server address structure
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    server_addr.sin_addr.s_addr = inet_addr(SERVER_ADDR);

    // Connect to server
    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Error connecting to server");
        exit(1);
    }

    // Send message to server
    write(sockfd, message, sizeof(message));

    // Close socket
    close(sockfd);
    return 0;
}
```

---

## Handling HTTP Requests

Many security tasks involve interacting with web servers, such as sending requests, retrieving data, or analyzing web traffic. The `libcurl` library is commonly used in C for making HTTP requests.

### Making HTTP Requests with `libcurl`

Here’s a simple example of how to use **libcurl** to send a GET request to a website:

```c
#include <stdio.h>
#include <curl/curl.h>

size_t write_callback(void *ptr, size_t size, size_t nmemb, char *data) {
    strcat(data, ptr);
    return size * nmemb;
}

int main() {
    CURL *curl;
    CURLcode res;
    char data[1024] = "";

    // Initialize curl
    curl_global_init(CURL_GLOBAL_DEFAULT);
    curl = curl_easy_init();

    if (curl) {
        // Set the URL to request
        curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");

        // Set the callback function to handle the response
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_callback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, data);

        // Perform the request
        res = curl_easy_perform(curl);

        // Check for errors
        if (res != CURLE_OK) {
            fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
        } else {
            // Output the received data
            printf("Response: %s\n", data);
        }

        // Cleanup
        curl_easy_cleanup(curl);
    }

    curl_global_cleanup();
    return 0;
}
```

### Parsing HTTP Responses

After receiving a response, you can parse it (e.g., extracting specific data like headers or content). In C, you may use regular expressions or string manipulation functions to process the response.

---

## Web Scraping in C

Web scraping involves extracting data from web pages, often in formats like HTML. C can be used for this task, but more complex tasks might require integrating other libraries like `libcurl` for fetching pages and a parser for HTML content.

### Basic Web Scraping with `libcurl` and `HTML Parsing`

While parsing HTML in C is complex, you can use libraries such as **libxml2** or **Gumbo**. Here’s a basic web scraping example that gets a page's content using `libcurl`:

```c
#include <stdio.h>
#include <curl/curl.h>

size_t write

_callback(void *ptr, size_t size, size_t nmemb, char *data) {
    strcat(data, ptr);
    return size * nmemb;
}

int main() {
    CURL *curl;
    CURLcode res;
    char data[1024] = "";

    // Initialize curl
    curl_global_init(CURL_GLOBAL_DEFAULT);
    curl = curl_easy_init();

    if (curl) {
        // Set the URL to scrape
        curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");

        // Set the callback function to handle the response
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_callback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, data);

        // Perform the request
        res = curl_easy_perform(curl);

        // Check for errors
        if (res != CURLE_OK) {
            fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
        } else {
            // Parse the HTML content here or print raw HTML for analysis
            printf("Scraped Data: %s\n", data);
        }

        // Cleanup
        curl_easy_cleanup(curl);
    }

    curl_global_cleanup();
    return 0;
}
```

This basic example fetches HTML content, but you would need an HTML parser to extract specific elements.

---

## Security Practices in C

Writing secure code in C is essential to prevent vulnerabilities like buffer overflows, uninitialized variables, and memory leaks. Here are some key practices:

### Buffer Overflow Protection

```c
#include <stdio.h>
#include <string.h>

void safe_copy(char *dest, const char *src) {
    strncpy(dest, src, 99);  // Limit the copy to 99 characters to prevent overflow
    dest[99] = '\0';         // Null-terminate the string
}

int main() {
    char buffer[100];
    char *input = "This is a safe copy!";

    safe_copy(buffer, input);
    printf("Buffer: %s\n", buffer);

    return 0;
}
```

### Cryptography with C

To implement encryption or hashing in C, you can use **OpenSSL**. Here’s an example of using **OpenSSL** to hash a string with **SHA-256**:

```c
#include <stdio.h>
#include <openssl/sha.h>

int main() {
    unsigned char hash[SHA256_DIGEST_LENGTH];
    char *message = "Hello, world!";

    // Compute the hash
    SHA256_CTX sha256_ctx;
    SHA256_Init(&sha256_ctx);
    SHA256_Update(&sha256_ctx, message, strlen(message));
    SHA256_Final(hash, &sha256_ctx);

    // Print the hash in hex format
    printf("SHA-256: ");
    for (int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
        printf("%02x", hash[i]);
    }
    printf("\n");

    return 0;
}
```

---

## Conclusion

C is a powerful tool for cybersecurity because of its **low-level access**, **performance**, and ability to interact with system resources directly. By mastering C and security concepts such as networking, HTTP requests, web scraping, and cryptography, you'll be able to develop tools, exploit vulnerabilities, and understand the inner workings of computer security.

---

## Resources

- [libcurl Documentation](https://curl.se/libcurl/)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Linux Sockets Programming](https://beej.us/guide/bgnet/)
- [C Programming Cheat Sheet](https://www.cprogramming.com/)
- [Learn Cybersecurity with C](https://www.learn-c.org/)

---

### Key Features:
- **Low-level Networking**: Learn to create servers and clients, manage TCP/UDP sockets.
- **Web Scraping**: Implement scraping tools using `libcurl` and basic HTML parsing techniques.
- **Security Libraries**: Use libraries like `libcurl`, `OpenSSL`, and `libpcap` to interact with networks, handle requests, and implement cryptography.
- **Real-world Security Practices**: Focus on secure coding practices like buffer overflow prevention and cryptography in C.

This guide provides an introduction to using C for cybersecurity tasks, with practical examples, useful libraries, and a deep dive into security features and best practices.
