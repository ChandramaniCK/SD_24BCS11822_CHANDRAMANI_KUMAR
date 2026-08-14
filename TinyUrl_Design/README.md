# TinyURL -- System Design

A scalable system design for a **TinyURL-style URL shortener**. The
architecture converts long URLs into compact short URLs and redirects
users to the original URL with low latency.

## 📌 Overview

The system provides two main operations:

1.  **Create Short URL** -- Accept a long URL and generate a unique
    short code.
2.  **Redirect** -- Accept a short URL and redirect the user to the
    original long URL.

The design also supports optional URL expiration, custom aliases,
caching, analytics, and horizontal scaling.

## 🏗️ Architecture

The main components are:

-   **Client / Browser** -- Sends URL creation and redirect requests.
-   **CDN / Edge Cache** -- Caches frequently accessed redirects close
    to users.
-   **Load Balancer** -- Distributes requests across application
    servers.
-   **API Gateway / Router** -- Routes requests to the appropriate
    service.
-   **Shortener Services** -- Stateless application servers responsible
    for creating and managing short URLs.
-   **Redirect Service** -- Handles short URL lookups and redirects.
-   **ID Generator** -- Generates unique IDs that are converted into
    Base62 short codes.
-   **Redis Cluster** -- Provides low-latency caching for short URL
    mappings.
-   **URL Mapping Database** -- Stores the permanent mapping between
    short codes and long URLs.
-   **Read Replicas** -- Scale read-heavy redirect traffic.
-   **Analytics Queue** -- Asynchronously processes click events.
-   **Analytics Store** -- Stores click and usage information.
-   **Admin / Analytics API** -- Provides access to analytics data.

## 🔄 Main Request Flows

### 1. Create a Short URL

``` text
Client
  ↓
Load Balancer
  ↓
API Gateway
  ↓
Shortener Service
  ↓
ID Generator
  ↓
Base62 Short Code
  ↓
Database + Redis
  ↓
Short URL returned to Client
```

Example:

``` text
Long URL:
https://example.com/products/category/item?id=12345

Generated short code:
aB72xQ1

Short URL:
https://tinyurl.com/aB72xQ1
```

### 2. Redirect a Short URL

``` text
Client
  ↓
CDN / Edge Cache
  ↓
Redirect Service
  ↓
Redis
  ↓
Database Read Replica (on cache miss)
  ↓
302 / 301 Redirect
  ↓
Original Long URL
```

Redis is checked first because redirects are read-heavy and require low
latency.

### 3. Analytics Flow

Analytics are handled asynchronously so that tracking clicks does not
significantly slow down redirects.

``` text
Redirect Service
      ↓
Analytics Queue
      ↓
Analytics Consumer
      ↓
Analytics Store
```

Possible analytics data includes:

-   Number of clicks
-   Timestamp
-   Referrer
-   User/device information
-   Geographic information, if collected

## 🔑 Short Code Generation

The system uses an ID generator such as a **Snowflake-style generator or
distributed counter**.

The generated numeric ID is converted into a **Base62** string.

Base62 uses:

``` text
A-Z
a-z
0-9
```

Therefore, a 7-character code provides:

``` text
62^7 ≈ 3.5 trillion
```

possible combinations.

This provides a very large namespace for short URLs.

## ⚡ Caching Strategy

Redis stores frequently accessed mappings:

``` text
shortCode → longURL
```

Example:

``` text
aB72xQ1 → https://example.com/products/item?id=12345
```

### Cache Hit

``` text
Request → Redis → Long URL → Redirect
```

### Cache Miss

``` text
Request → Redis → Database → Redis → Redirect
```

This reduces database load and improves redirect latency.

## 🗄️ Database Design

A basic URL mapping table can contain:

  Column         Description
  -------------- ---------------------------
  `short_code`   Unique short identifier
  `long_url`     Original URL
  `created_at`   Creation timestamp
  `expires_at`   Optional expiration time
  `user_id`      Optional owner/user ID
  `is_active`    Whether the URL is active

The `short_code` should have a unique index.

## 📊 Analytics Design

Analytics should not block the redirect request.

Instead:

``` text
Redirect
   ↓
Publish Event
   ↓
Queue
   ↓
Consumer
   ↓
Analytics Store
```

This makes analytics **eventually consistent**, while the core
URL-shortening and redirect functionality remains highly available.

## 📈 Capacity Estimates

Example assumptions:

-   DAU: **10 million**
-   Short URLs created per user per day: **2**
-   Approximate writes: **20 million/day**
-   Redirects per short URL per day: **10**
-   Approximate reads: **200 million/day**

The system should therefore be optimized for a **read-heavy workload**.

The architecture uses:

-   Redis
-   CDN
-   Read replicas
-   Horizontal application scaling

to handle the high redirect volume.

## 🔒 Important Design Considerations

### Unique Short URLs

The generated short code must be globally unique.

A distributed ID generator avoids collisions when multiple application
servers create URLs simultaneously.

### Horizontal Scaling

Application servers are stateless, so additional instances can be added
behind the load balancer.

``` text
             ┌─ Service 1
Load Balancer ├─ Service 2
             └─ Service 3
```

### High Availability

The system avoids depending on a single application server.

Database replication, Redis clustering, and multiple application
instances improve availability.

### Expiration

If an URL has an expiration time, the redirect service can check:

``` text
currentTime < expiresAt
```

If expired, return an appropriate error instead of redirecting.

### Abuse Protection

A production implementation should also consider:

-   Rate limiting
-   Malicious URL detection
-   Spam prevention
-   Authentication for management APIs
-   URL validation
-   Access controls

## 📁 Files

``` text
ShortUrl/
├── TinyURL System Design.md
├── TinyURL_System_Design_Modified.drawio
└── README.md
```

The `.drawio` file contains the editable system architecture diagram.

## 🎯 Design Goals

The architecture is designed to provide:

-   **Low-latency redirects**
-   **High availability**
-   **Horizontal scalability**
-   **Globally unique short URLs**
-   **Reliable URL storage**
-   **Efficient caching**
-   **Asynchronous analytics**
-   **Read scalability through replicas**

## 🧩 Technology Choices

Possible technologies for implementation:

  Component         Example Technology
  ----------------- -----------------------------------------
  API / Backend     Java + Spring Boot
  Load Balancer     Nginx / Cloud Load Balancer
  Cache             Redis
  Database          MySQL / PostgreSQL
  Message Queue     Kafka / AWS SQS
  Analytics Store   Elasticsearch / ClickHouse / PostgreSQL
  Deployment        Docker + Kubernetes
  CDN               Cloudflare / AWS CloudFront

These technologies are interchangeable; the architecture is based on the
responsibilities of each component rather than a specific vendor.

## 🚀 Future Improvements

Possible extensions include:

-   Custom aliases
-   URL expiration
-   User authentication
-   QR code generation
-   Rate limiting
-   Geographic routing
-   Multi-region deployment
-   Database sharding
-   Bloom filters for invalid URL detection
-   Advanced analytics dashboards
-   Distributed tracing and monitoring

------------------------------------------------------------------------

## 📌 Summary

The TinyURL system follows a **cache-first, read-optimized
architecture**.

The most important flow is:

``` text
Client
  ↓
CDN / Load Balancer
  ↓
Redirect Service
  ↓
Redis
  ↓
Database Replica (cache miss)
  ↓
301/302 Redirect
```

For URL creation:

``` text
Client
  ↓
API Gateway
  ↓
Shortener Service
  ↓
ID Generator → Base62
  ↓
Database + Redis
  ↓
Short URL
```

This design provides a good foundation for building a production-scale
URL-shortening service.
