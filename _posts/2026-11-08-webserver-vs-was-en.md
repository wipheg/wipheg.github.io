---
title: "Web Server vs WAS, and Apache vs Nginx"
date: 2026-11-08 09:03:00 +0900
categories: [Backend, Infra]
tags: [web-server, was, apache, nginx, tomcat]
description: What a web server and a WAS each do, why they are usually run separately, and how Apache and Nginx differ in design and where each one fits.
toc: true
lang: en
ref: webserver-vs-was
hidden: true
---

If you study backend development, you keep running into "web server", "WAS", "Apache", "Nginx" and "Tomcat". They all accept HTTP requests, so they look alike. The difference is that a web server sends back files that already exist, while a WAS runs code to build a new response.

> WAS stands for Web Application Server. The term is common in Korea. In English you will more often see "application server" or "servlet container" for the Java case.
{: .prompt-info }

> **Summary**
> - A web server serves static files and forwards requests. A WAS runs application code to build dynamic responses.
> - A WAS can run a service on its own, but a web server usually sits in front of it for performance, security, scaling and fault isolation.
> - Apache gives each connection its own process or thread. Per-directory config (`.htaccess`) and modules make it easy to add features.
> - Nginx is event-driven, so it handles many concurrent connections with few resources. It is widely used as a reverse proxy and load balancer.
{: .prompt-tip }

## 1. Static and dynamic content

What separates a web server from a WAS is whether the content it handles is static or dynamic.

| | Static content | Dynamic content |
| --- | --- | --- |
| What it is | A file that already exists | A result built by running logic on each request |
| Examples | HTML, CSS, JS, images, fonts | Logged-in user info, search results, order history |
| Behavior | Same response for everyone | Changes with the request, the user and the DB state |

For static content, the web server reads the file from disk and sends it as is. For dynamic content, the WAS runs code to produce it.

## 2. Web server

A web server takes an HTTP request and sends back a static file unchanged. Requests that need dynamic processing are passed on to the WAS behind it.

A web server does five main jobs.

- Serving static files (HTML, CSS, JS, images)
- Reverse proxy: passing client requests to a backend server (the WAS)
- Load balancing: spreading requests across several WAS instances
- SSL/TLS termination: handling HTTPS encryption and decryption
- Caching, gzip compression, access control, rate limiting

Common examples are Apache HTTP Server, Nginx, Caddy, LiteSpeed and IIS.

## 3. WAS

A WAS (Web Application Server) runs application code such as DB queries and business logic to build dynamic content. In Java, this is the servlet container that runs servlets and JSPs.

A WAS does four main jobs.

- Running business logic
- Talking to the DB and handling transactions
- Managing sessions
- Managing thread pools and connection pools

On the Java side, examples are Tomcat, Jetty, Undertow, WildFly (JBoss), WebLogic and JEUS. Gunicorn and uWSGI for Python, and PHP-FPM for PHP, play a similar role.

> Tomcat includes an HTTP connector (Coyote), so it can take browser requests directly and serve static files too. So why not run Tomcat alone without Nginx?
{: .prompt-info }

## 4. Why run them separately?

A WAS can run a service by itself. Even so, most production setups put a web server in front of it.

```text
Client ──▶ Web Server (Nginx) ──▶ WAS (Tomcat) ──▶ DB
              │
              └─ static files are answered here
```

**Splitting the load.** Tomcat handles one request on one thread, and the number of threads has an upper limit (200 by default for embedded Tomcat in Spring Boot). If Tomcat also takes requests for images and CSS, those threads are tied up sending files, and requests that need real logic have to wait for a free thread. Let the web server send static files and keep Tomcat threads for requests that need logic.

**Security.** Only the web server's port is open to the outside. The WAS sits behind it on the internal network, where clients can't reach it directly.

**Load balancing and zero-downtime deploys.** One web server spreads requests across several WAS instances. During a deploy, you take WAS instances out and put them back one at a time, so the service never stops.

**Fault isolation.** If the WAS goes down, the web server is still up and can show a maintenance or error page.

**SSL, compression and caching in one place.** The HTTPS certificate is installed only on the web server, and the web server talks to the WAS over plain HTTP on the internal network. Even with several WAS instances, there is only one place to renew the certificate.

> Does embedded Tomcat in Spring Boot make Nginx unnecessary? For a single instance serving plain HTTP, you can skip it. Once you need HTTPS, load balancing across instances, zero-downtime deploys or static file caching, the usual move is to put Nginx or a cloud load balancer in front.
{: .prompt-tip }

## 5. Web server vs WAS

| | Web server | WAS |
| --- | --- | --- |
| Main job | Serve static content, forward requests | Build dynamic content |
| Handles | HTML, CSS, JS, images | Business logic, DB access |
| Examples | Nginx, Apache HTTP Server | Tomcat, Jetty, Undertow |
| Position | In front, closest to the client | Behind the web server, internal network |
| Runtime | Native program written in C | Runs on a language runtime such as the JVM or Python |

## 6. Apache and Nginx

The two most widely used web servers are Apache HTTP Server and Nginx.

> "Apache" usually means Apache HTTP Server (httpd). Tomcat is also an Apache Software Foundation project and is called "Apache Tomcat", but the two are completely different pieces of software.
{: .prompt-warning }

### 6-1. Background

Apache came out in 1995 and was the most used web server for a long time. You add features like URL rewriting, SSL and authentication by loading modules.

Nginx was released in 2004. It was built to solve the C10K problem: handling 10,000 concurrent connections on a single server. Web server usage statistics such as W3Techs now put Nginx ahead of Apache.

### 6-2. How requests are handled

This is where the differences in concurrent connection capacity and memory use come from.

**Apache is process/thread based.** Each connection is handled by one process or thread. You choose the model with an MPM (Multi-Processing Module).

| MPM | How it works | Notes |
| --- | --- | --- |
| prefork | 1 connection = 1 process | Stable, uses a lot of memory, works with mod_php |
| worker | Several threads per process | Uses less memory than prefork |
| event | worker + Keep-Alive connections handled separately | Idle connections cost less, recommended in 2.4 |

```text
Request A ──▶ [Thread 1]
Request B ──▶ [Thread 2]
Request C ──▶ [Thread 3]   ← more connections, more threads and memory
```

Because each request is tied to one process or thread, the flow is easy to follow. With prefork, if one process dies, requests handled by other processes are not affected. The downside is that 10,000 concurrent connections need about as many processes or threads, which drives up memory use and context switching.

**Nginx is event-driven, asynchronous and non-blocking.** It runs one master process and a few worker processes (usually one per CPU core). Each worker runs an event loop and handles thousands to tens of thousands of connections at once.

```text
             ┌─ Request A
[Worker 1] ──┼─ Request B    ← handles whichever event is ready, serves others while waiting
             └─ Request C ...
```

No thread is parked on a connection that is waiting for a response, so memory barely grows even with tens of thousands of open connections. That is why Nginx does well at jobs that spend most of their time waiting, like sending static files and proxying. A long blocking operation inside a worker, though, stalls every connection that worker owns.

### 6-3. Dynamic content

Apache can run PHP inside the server itself with a module like `mod_php`. Drop a PHP file in place and Apache runs it right away, with no separate process to start.

Nginx does not process dynamic content itself. It hands it to an external process.

- PHP → PHP-FPM (FastCGI)
- Java → Tomcat (`proxy_pass`)
- Python → Gunicorn, uWSGI

You have to start a separate process like PHP-FPM and configure the connection to it. In return, you can restart and scale the web server and the application independently.

### 6-4. Configuration

| | Apache | Nginx |
| --- | --- | --- |
| Per-directory config | Can override with `.htaccess` | None, everything in the main config |
| Pro | Change settings without admin access | No file lookups, config in one place |
| Con | Directory lookups on every request | Changes need server access |

### 6-5. Modules

Apache can load modules at runtime and has a very large number of them (`mod_rewrite`, `mod_ssl`, `mod_security` and more).

Nginx originally required modules to be compiled in. It has supported dynamic modules since 1.9.11, but its module ecosystem is smaller than Apache's.

## 7. Apache vs Nginx

| | Apache | Nginx |
| --- | --- | --- |
| Design | Process/thread based (MPM) | Event-driven, async non-blocking |
| Concurrency | Resources grow with connections | Many connections on few resources |
| Idle connections | Hold a thread (less so with event MPM) | Handled by the event loop, low cost |
| Dynamic content | Can run it directly via modules | Hands it to an external process |
| Configuration | Per-directory `.htaccess` | Main config file only |
| Modules | Loaded at runtime, many | Dynamic since 1.9.11, fewer |
| Typical use | Legacy, shared hosting, PHP | Reverse proxy, load balancer, high traffic |

> With the event MPM, Apache also hands connections that are only being kept alive to a separate thread. Worker threads get stuck on idle connections far less often than before. So the common claim that Nginx is always faster is too simple. Nginx is easier for heavy concurrency and proxying, and Apache is easier for per-directory config and module extensions.
{: .prompt-info }

## 8. Which one to use

**When to pick Nginx**

- Services with heavy traffic and many concurrent connections
- A reverse proxy in front of apps like Spring Boot, Node.js or Django
- Load balancing, SSL termination, serving static files
- Container or cloud environments (one config file covers the proxy setup)

**When to pick Apache**

- Shared hosting where `.htaccess` is required
- A traditional PHP-based LAMP stack such as WordPress
- Legacy systems already running on Apache
- When you specifically need an Apache module

**Using both**

Some setups put Nginx in front for static files and proxying, with Apache behind it for dynamic processing. People do this when they want to keep the existing Apache config and `.htaccess` files and move only static files and SSL to Nginx.

```text
Client ──▶ Nginx (static files, SSL, proxy) ──▶ Apache + PHP
```

## 9. Example: Nginx + Spring Boot reverse proxy

Here is a minimal config that puts Nginx in front of two Spring Boot instances (ports 8080 and 8081).

```nginx
upstream app_servers {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;   # load balance across 2 instances
}

server {
    listen 80;
    server_name example.com;

    # Nginx serves static files directly
    location /static/ {
        root /var/www;
        expires 7d;
    }

    # everything else goes to the WAS
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> Behind Nginx, the client IP the WAS sees is always Nginx's IP (127.0.0.1). The real client IP has to be passed in the `X-Forwarded-For` header as above, and Spring Boot only reads that header once `server.forward-headers-strategy` is set.
{: .prompt-warning }

## Wrap-up

- A web server sends files back as they are and forwards requests. A WAS runs code to build the response.
- A WAS can run alone, but a web server goes in front to save WAS threads, keep the WAS off the public network, spread load across instances and show an error page when the WAS is down.
- Apache uses a process or thread per connection. It is common on shared hosting that needs `.htaccess` and in existing PHP environments.
- Nginx uses a few workers with event loops to handle tens of thousands of connections. It is widely used as a reverse proxy and load balancer.
- For a new service, the usual starting point is a WAS behind Nginx.

## References

- [Apache HTTP Server 2.4: Multi-Processing Modules (MPMs)](https://httpd.apache.org/docs/2.4/mpm.html)
- [Apache HTTP Server 2.4: .htaccess files](https://httpd.apache.org/docs/2.4/howto/htaccess.html)
- [nginx documentation](https://nginx.org/en/docs/)
- [nginx: Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [The C10K problem](http://www.kegel.com/c10k.html)
