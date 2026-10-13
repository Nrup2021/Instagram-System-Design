# Instagram-System-Design
Deliver the right content to the right user as quickly and reliably as possible, even when the system handles a huge amount of traffic.

## System Architecture Diagram

The diagram covers the client apps, load balancer, application servers, cache, sharded databases, S3 object storage, CDN, and the four caching levels.

<img width="1536" height="1024" alt="Instagram System Design Architecture Infographic" src="https://github.com/user-attachments/assets/6a478e09-6531-4b81-86e6-b9f532236f48" />

## 1. Introduction

Instagram is a social media application where users can create profiles, upload photos and videos, follow other users, view feeds, like posts, comment, and share content.

Designing a system like Instagram requires handling:

- Millions of users and their relationships.
- Large volumes of images and videos.
- High read traffic for news feeds.
- Fast content delivery across different countries.
- Database scaling and caching.
- Mobile data usage and network latency.

## 2. MVP — Minimum Viable Product

An MVP is the simplest version of a product that provides its core functionality to users.

### Instagram MVP example

Initially, Instagram might support only:

1. User registration and login.
2. Creating a profile.
3. Uploading photos with captions.
4. Following other users.
5. Viewing a feed.
6. Liking and commenting on posts.

Features such as advanced recommendations, video editing, stories, and sophisticated analytics can be introduced later.

**Real-world scenario:** Instead of building every Instagram feature from day one, launch a basic application, collect feedback, and improve it gradually.

## 3. Important System Design Concepts

### QPS — Queries Per Second

QPS measures how many requests a system processes per second.

**Example:** If users make 100,000 requests in 10 seconds:

QPS = 100,000 / 10 = 10,000

The system must be designed to handle the expected request volume, including peak traffic.

Instagram requests might include loading a feed, opening a profile, liking a post, or fetching comments.

### Latency

Latency is the time taken for a request to receive a response.

**Example:** When you open Instagram, the feed should appear quickly rather than making you wait several seconds.

Ways to reduce latency:

- Cache frequently accessed data.
- Use a CDN for images and videos.
- Optimize database queries and indexes.
- Reduce unnecessary network requests.
- Load content asynchronously.
- Fetch only the data the user needs.

### RTT — Round-Trip Time

RTT is the time taken for a request to travel from the client to a server and for the response to return.

For example:

1. The Instagram app requests a user's profile.
2. The request travels to the server.
3. The server processes the request.
4. The response returns to the app.

The total network round trip contributes to the perceived response time.

A server located far from the user may have a higher RTT, which is one reason geographically distributed CDN servers are useful.

### CAP Theorem

CAP stands for:

- **Consistency:** Clients see data that respects the system's consistency guarantees.
- **Availability:** Requests to a functioning system receive responses.
- **Partition Tolerance:** The system continues operating despite certain network communication failures between distributed nodes.

The CAP theorem states that when a network partition occurs, a distributed system must make a trade-off between consistency and availability for the affected operations.

**Instagram example:** A like count might briefly differ between two screens because updates are being propagated. That may be acceptable for some feed interactions, whereas authentication and certain account operations may require stronger consistency.

CAP is not simply a choice between consistency and availability under normal conditions; the trade-off becomes important during a network partition.

### API — Application Programming Interface

An API allows the mobile application or browser to communicate with backend services.

Example endpoints:

- `POST /users` — Create a user.
- `GET /users/123` — Fetch a user profile.
- `POST /posts` — Create a post.
- `GET /feed` — Fetch the news feed.
- `POST /posts/456/likes` — Like a post.
- `GET /posts/456/comments` — Fetch comments.

The mobile application sends requests, and the backend returns data, often in JSON format.

## 4. Instagram's Core Features

### User Profile

A profile contains information such as:

- User ID and username.
- Profile picture and biography.
- Followers and following.
- Posts created by the user.

### Posts and Media

A post can contain text, an image, or a video.

A post's metadata might include:

- Post ID.
- Creator ID.
- Caption.
- Creation timestamp.
- Media URL or storage key.
- Media type.
- Like count and comment count.

**Important:** Media files and post metadata should generally be stored separately. Images and videos are large binary files, while metadata is comparatively small and can be stored in a database.

### Feed

The feed displays posts from accounts the user follows, usually with the newest posts near the top.

### Likes and Comments

Users can interact with posts by liking them and adding comments.

The backend must store these interactions and update the information shown to users.

### Share

Users can share a post with other users or through an external sharing mechanism.

The system needs to identify the original post and control access according to the application's privacy rules.

## 5. Database Design

A simplified relational database design could look like this.

### Users table

| Column | Purpose |
|---|---|
| `user_id` | Unique user identifier |
| `name` | User's name |
| `dob` | Date of birth |
| `gender` | Gender, if collected |

### User_Follower table

| Column | Purpose |
|---|---|
| `user_id` | Account being followed |
| `follower_id` | Account that follows the user |

Example:

If Alice follows Bob:

- `user_id = Bob`
- `follower_id = Alice`

This table stores relationships between users.

### Posts table

| Column | Purpose |
|---|---|
| `post_id` | Unique post identifier |
| `creator_id` | User who created the post |
| `created_at` | Post creation timestamp |
| `like_count` | Number of likes |
| `comment_count` | Number of comments |
| `media_url` | Media location or reference |
| `media_type` | Image, video, or other supported type |

In a production system, the media reference might be an object-storage key rather than a publicly accessible URL.

### Post_Like table

| Column | Purpose |
|---|---|
| `user_id` | User who liked the post |
| `post_id` | Post that was liked |

A unique constraint on `(user_id, post_id)` can prevent the same user from liking the same post more than once.

### Post_Comments table

| Column | Purpose |
|---|---|
| `comment_id` | Unique comment identifier |
| `post_id` | Post receiving the comment |
| `user_id` | User who wrote the comment |
| `comment` | Comment text |
| `created_at` | Comment timestamp |

A `comment_id` is useful because one user can write many comments on the same post.

**Database design takeaway:** Use relationships to connect users, posts, likes, and comments. Add indexes to support the queries that are executed most frequently.

## 6. How to Fetch the Instagram Feed Using SQL

Suppose Alice follows Bob, Carol, and David. When Alice opens Instagram, the system needs to retrieve recent posts created by those accounts.

A simplified SQL query is:

```sql
SELECT p.*
FROM User_Follower AS uf
JOIN Posts AS p
    ON p.creator_id = uf.user_id
WHERE uf.follower_id = 100
ORDER BY p.created_at DESC
LIMIT 50;
```

Here, user `100` is the person viewing the feed.

### How this query works

1. Find the accounts followed by user 100.
2. Find posts created by those accounts.
3. Sort the posts by creation time, newest first.
4. Return up to 50 posts.

For a real production feed, additional filtering, ranking, privacy checks, pagination, and deduplication may be required.

### Why database indexes matter

An index can help the database locate matching records without scanning the entire table.

Useful index candidates include:

- `User_Follower(follower_id, user_id)`
- `Posts(creator_id, created_at)`
- `Post_Like(post_id, user_id)`
- `Post_Comments(post_id, created_at)`

Indexes should be selected based on actual query patterns and measured performance.

## 7. Scaling the Database — Sharding

Imagine Instagram stores around 73 TB of data. This is an illustrative scale example from the class notes, not a fixed Instagram storage figure.

One database server may not provide enough storage, memory, or processing capacity to handle all the data and requests efficiently.

**Sharding** divides data across multiple database servers, called shards.

### User-based sharding

One approach is to distribute users according to their user IDs.

Example:

- Shard A stores a group of users and their associated data.
- Shard B stores another group.
- Shard C stores another group.

The precise assignment depends on the chosen sharding strategy.

### Consistent hashing

Consistent hashing is a technique for distributing keys across multiple servers while minimizing how many keys must move when servers are added or removed.

**Real-world scenario:** If Instagram adds another database server, consistent hashing can help redistribute a portion of the data rather than moving everything.

### Important challenge

A feed may contain posts from users stored on different shards. The application must collect results from those shards and combine them efficiently.

Sharding helps with scale, but it also adds complexity to queries, transactions, and data distribution.

## 8. Caching Recent Posts

A database query for every feed request can become expensive when many users repeatedly request similar data.

Caching stores frequently accessed information in a faster location.

For example, a system might cache recent post metadata from the last 10–15 days. This is an illustrative design choice from the notes, not a universal retention period.

Cached metadata could include:

- Post ID.
- Creator ID.
- Creation time.
- Media reference.
- Like and comment counts.

When a user requests the feed, the system can retrieve recent metadata from the cache and avoid repeating some database operations.

### Why cache recent posts?

Recent posts are often accessed more frequently than old posts.

A cache can reduce database load and improve response time, provided the system handles expired or outdated entries correctly.

### Example query

```sql
SELECT *
FROM Posts
WHERE creator_id IN (101, 102, 103)
ORDER BY created_at DESC
LIMIT 100;
```

This retrieves recent posts from a specified group of creators, subject to the database's execution plan and schema.

**Important:** Caching the results can improve performance, but a query like this alone does not guarantee low latency. Indexing, query design, cache hit rate, and the amount of data processed also matter.

## 9. High-Level Instagram Architecture

A simplified architecture for retrieving a feed and displaying its media looks like this:

```text
Mobile App / Browser
        |
        v
   Load Balancer
        |
        v
   Application Servers
        |
        +------------------+
        |                  |
        v                  v
   Cache Layer       Database Shards
        |
        v
  Cached Feed Metadata


Mobile App / Browser
        |
        v
     CDN Edge
        |
        | Cache Miss
        v
  Origin Object Storage
      (e.g. S3)
```

### What each component does

**Client:** The Android app, iOS app, or browser used by the customer.

**Load balancer:** Distributes incoming requests across application servers.

**Application servers:** Handle business logic, authorization, feed generation, and API responses.

**Cache layer:** Stores frequently accessed data for faster retrieval.

**Database shards:** Store structured data such as users, posts, and relationships.

**Object storage:** Stores the actual image and video files.

**CDN:** Delivers cached media from servers located closer to users.

Consistent hashing may be used in certain parts of the system to distribute data or requests. It is not a mandatory replacement for a load balancer.

### Database technology choices

- **MySQL:** A relational database suitable for structured records and relationships.
- **Cassandra:** A distributed database designed for high-volume workloads with appropriate data models.
- **MongoDB:** A document database that stores data in flexible document structures.
- **Amazon S3:** Object storage suitable for large media files.

A large application may use different technologies for different workloads. There is no need to put all data into a single database technology.

## 10. Handling Images and Videos

Photos and videos are much larger than ordinary text or metadata. Sending every media file directly from an application server can waste resources and increase latency.

A better approach is to store the media separately and serve it through a CDN.

### Step 1: Upload media

When a user uploads a photo:

1. The client uploads the media through the supported upload flow.
2. The backend validates the request and permissions.
3. The image or video is stored in object storage such as Amazon S3.
4. The system saves the media reference and post metadata.
5. The post becomes available in the appropriate feeds.

The actual implementation may use direct-to-storage uploads with temporary authorized URLs to avoid routing large files through application servers.

### Step 2: Save metadata

The database stores information such as:

```text
post_id: 456
creator_id: 100
created_at: 2026-10-13T10:30:00Z
media_type: image
media_key: media/abc123
```

The media key identifies the stored object. It is not necessarily the final URL displayed to a user.

### Step 3: Fetch media

When another user opens the post:

1. The app obtains the media reference.
2. It requests the image through the CDN URL.
3. If the CDN has the requested object cached, it serves the image.
4. If the object is not cached, the CDN retrieves it from the configured origin, such as S3, when permitted.
5. The CDN can cache the response for subsequent requests.

**Key concept:** The CDN is primarily a content-delivery and caching layer, not the permanent source of truth or a backup system.

## 11. CDN — Content Delivery Network

A CDN is a geographically distributed network of servers that delivers content closer to the user.

Examples:

- Akamai
- Cloudflare
- Amazon CloudFront

These providers operate distributed infrastructure in multiple regions and locations.

### POP — Point of Presence

A POP is a location where a CDN provides network infrastructure and content-delivery services.

CDN edge servers at or associated with these locations help serve content closer to users.

### Real-world example

Suppose an image is stored in an AWS S3 bucket in one region, but a user in Europe requests it.

Without a CDN, the image may need to travel from the origin region to Europe for each uncached request.

With a CDN:

1. The browser requests the image from a CDN URL.
2. The CDN checks whether the image is cached at an appropriate edge location.
3. On a cache hit, the edge serves the image.
4. On a cache miss, the CDN fetches it from the origin if configured to do so.
5. The CDN may retain a cached copy for future requests.

This reduces repeated origin requests and can lower latency.

### How does the CDN know which image to serve?

The client requests a specific URL or cache key.

For example:

```text
https://cdn.example.com/media/abc123.jpg
```

The identifier `abc123` could represent an opaque object key. It might be derived from a generated ID or hash, depending on the system.

The CDN uses the request URL and its cache configuration to determine which content to return. It does not randomly choose an image.

### CDN versus object storage

| Component | Main responsibility |
|---|---|
| S3 or other object storage | Persist the original media object |
| CDN | Cache and deliver content efficiently |
| Database | Store post metadata and media references |
| Application server | Enforce rules and handle application logic |

A CDN can cache several image variants for different resolutions when the system is configured to generate or serve them.

## 12. Browser and Mobile Performance Optimization

### Asynchronous loading

A browser or mobile app should avoid waiting for every image before displaying the rest of the feed.

For example, the app can display post text and image placeholders immediately while images load in the background.

### Lazy loading

Lazy loading means loading images when they are approaching the visible screen area instead of downloading every image at once.

**Example:** If a feed contains 100 posts but only 5 are visible, the app can prioritize those 5 images and load others as the user scrolls.

### Placeholder images

A placeholder is displayed before the actual media is available.

It gives users immediate visual feedback instead of showing a blank area.

### Shimmer effect

A shimmer effect is an animated loading indicator, often used while content is being fetched.

It communicates that content is still loading.

### Responsive image resolution

The application can select an appropriate image variant based on the display size and network conditions.

Example variants:

- 240p
- 360p
- 480p
- 1024p
- 4K

These are illustrative resolution labels, not a universal image-sizing standard. For images, pixel dimensions are usually more informative than video-style `p` labels.

**Real-world scenario:** A small thumbnail in the feed does not need to download the same high-resolution image used when a user opens the full-screen viewer.

Serving a suitably sized image reduces transfer time, bandwidth use, and memory consumption.

For video, adaptive bitrate streaming can select an appropriate quality based on device capabilities and network conditions.

## 13. Mobile Caching and Delta Fetch

Mobile applications should avoid downloading the same data every time a user opens the app.

They can maintain local copies of feed metadata and media, with suitable expiration and invalidation rules.

### Local caching

A mobile application can store feed metadata in a local database such as SQLite or Room on Android.

It can also cache images using a dedicated image-loading library.

Benefits include:

- Faster feed rendering.
- Reduced repeated network requests.
- Lower data usage.
- Better perceived performance during temporary connectivity problems.

Local caching must still respect privacy, account changes, and content deletion.

### Delta fetch

Delta fetch means retrieving only the changes since the last successful synchronization instead of downloading the entire dataset again.

Suppose the app last synchronized at 10:00 AM.

At 10:05 AM, the user opens Instagram again. Instead of downloading the entire feed history, the client can request updates after its previous synchronization point.

Conceptually:

```http
GET /feed/updates?since=2026-10-13T10:00:00Z
```

The server returns the changes available after that point, according to the API's synchronization rules.

The actual implementation might use a cursor, sequence number, or opaque synchronization token rather than a timestamp.

**Important:** Delta fetch must account for deleted posts, edits, visibility changes, and updates that arrive out of order. The server should define how clients learn about these changes.

### Pagination

Pagination divides a large result set into smaller chunks.

For example:

- First request: 20 posts.
- Next request: the next 20 posts.
- Further requests: additional posts as the user scrolls.

Cursor-based pagination is often preferable for feeds because new posts can be inserted while the user is scrolling.

A cursor might identify the last item received rather than using a simple page number.

### Delta fetch versus pagination

| Concept | Purpose |
|---|---|
| Pagination | Retrieve a large result set in manageable chunks |
| Delta fetch | Retrieve changes since a previous synchronization |
| Local caching | Reuse previously downloaded data on the device |
| CDN caching | Reuse cached media closer to the user |
| Database caching | Reduce repeated database work |

These mechanisms solve related but different problems.

## 14. Cache Invalidation and Deleted Content

Cache invalidation means removing or updating cached data when it is no longer valid.

Suppose a user deletes a post.

The system may need to:

1. Mark the post as deleted in the authoritative database.
2. Prevent the post from appearing in future feed responses.
3. Invalidate or update cached feed metadata.
4. Purge the relevant CDN object or image variants when appropriate.
5. Notify or synchronize changes with clients that have cached copies.
6. Remove local cached data when the app learns about the deletion.

### CDN invalidation

If a CDN has cached an image, deleting the original from S3 does not necessarily remove the cached copy immediately.

Depending on the CDN configuration, the system may need to issue an invalidation request, use short cache lifetimes, or use versioned URLs.

### Mobile cache invalidation

The mobile app should also remove or update local records when it learns that content has been deleted or access has changed.

For private or sensitive content, access control should not depend solely on cache expiration.

## 15. Four Layers of Caching — Class Notes Summary

The following is a useful learning model for remembering the different optimization techniques. The numbering is a study aid, not a universal industry-standard classification.

### Level 1: Recent post metadata cache

Cache frequently accessed feed metadata to reduce database queries.

### Level 2: CDN media cache

Serve images and videos from geographically distributed CDN edge locations.

### Level 3: Browser or mobile local cache

Reuse previously downloaded images and locally stored feed information.

### Level 4: Delta fetch and synchronization

Request only the updates needed to bring the local feed closer to the server's current state.

Together, these techniques can reduce database load, network traffic, and perceived latency.

## 16. Complete Example — What Happens When You Open Instagram?

Imagine Alice opens Instagram on her phone.

1. **Open the app:** The app displays locally cached content if available.
2. **Fetch updates:** It requests the next feed page or changes since its last synchronization point.
3. **Reach the backend:** The request passes through the appropriate network and load-balancing infrastructure.
4. **Retrieve feed data:** Application services use caches and databases to obtain posts from accounts Alice follows.
5. **Return metadata:** The API sends post details, captions, timestamps, and media references.
6. **Display the feed:** The app renders text and placeholders immediately where possible.
7. **Fetch media:** Image requests go to the CDN.
8. **Check the CDN cache:** The edge serves cached media or fetches a cache miss from the origin.
9. **Load images efficiently:** The app prioritizes visible images and suitable resolutions.
10. **Cache locally:** Images and selected data may be reused on later visits.
11. **Handle interactions:** When Alice likes or comments on a post, the backend processes the action and updates the relevant data.
12. **Synchronize changes:** Subsequent feed requests retrieve new content and relevant updates.

The result is a feed that can load quickly without downloading every historical post or media file on every visit.

## 17. Key System Design Takeaways

- **MVP:** Start with essential features before expanding the product.
- **Low latency:** Optimize the time users wait for requests and content.
- **Database indexes:** Make frequent queries more efficient.
- **Sharding:** Distribute large datasets across multiple database servers.
- **Consistent hashing:** Distribute keys while minimizing redistribution when the server pool changes.
- **Recent metadata caching:** Reduce repeated database work for frequently requested feed data.
- **Object storage:** Keep large media files separate from structured metadata.
- **CDN:** Deliver media from locations closer to users.
- **Lazy loading:** Fetch images when they are needed.
- **Responsive media:** Avoid downloading unnecessarily large image variants.
- **Local caching:** Reuse data and media already available on the device.
- **Delta fetch:** Download changes instead of repeatedly fetching the full dataset.
- **Pagination:** Retrieve feed items in manageable chunks.
- **Cache invalidation:** Keep cached data consistent with important changes, including deletions.

---

# 18. Quick Glossary — Short Lines for Revision

| Term | Easy meaning |
|---|---|
| **MVP** | Minimum Viable Product — the first version with essential features. |
| **QPS** | Queries Per Second — requests processed per second. |
| **Latency** | Time taken for a request to receive a response. |
| **CAP** | Consistency, Availability, Partition Tolerance — trade-offs in distributed systems during network partitions. |
| **API** | A defined interface that lets software systems communicate. |
| **RTT** | Round-Trip Time — time for a request and response to travel between client and server. |
| **CDN** | Content Delivery Network — serves content from distributed locations. |
| **POP** | Point of Presence — a location where a CDN provides network infrastructure. |
| **Sharding** | Splitting a dataset across multiple database servers. |
| **Consistent hashing** | A key-distribution technique that minimizes data movement when servers change. |
| **Cache** | A faster storage layer for frequently accessed data. |
| **Cache hit** | The requested item is available in the cache. |
| **Cache miss** | The requested item is not available in the cache. |
| **Cache invalidation** | Removing or updating cached data that is no longer valid. |
| **Metadata** | Information describing a post, such as its ID, creator, and timestamp. |
| **Object storage** | Storage designed for files such as images and videos. |
| **S3** | Amazon's object storage service. |
| **Database index** | A data structure that helps the database find records efficiently. |
| **Lazy loading** | Loading content only when it is needed or approaching visibility. |
| **Asynchronous loading** | Loading content without blocking unrelated work. |
| **Placeholder** | Temporary content displayed while the real content loads. |
| **Shimmer effect** | An animated visual indicator that content is loading. |
| **Responsive images** | Image variants selected to suit the display and usage needs. |
| **Pagination** | Retrieving data in smaller chunks instead of all at once. |
| **Delta fetch** | Retrieving changes since the previous synchronization. |
| **Local cache** | Data stored on the user's device for reuse. |
| **Feed** | A collection of posts presented to a user. |
| **Eventual consistency** | A model in which replicas may temporarily differ but converge if updates stop and propagation succeeds. |
| **Load balancer** | Distributes incoming requests across available servers. |
| **Origin server** | The source from which a CDN retrieves content on a cache miss. |
| **Edge server** | A server positioned closer to users to deliver content efficiently. |
| **80/20 rule** | A general heuristic that a minority of items may account for most activity; the exact proportions vary. |
| **80–19–1 rule** | A rough participation heuristic: 80% mostly consume, 19% contribute occasionally, and 1% create frequently. It is not a universal law. |

## Final Interview Revision

**Question: How would you design Instagram's feed to load faster?**

I would optimize the feed at multiple levels. First, I would index the database queries and cache frequently accessed post metadata. I would use pagination or cursor-based retrieval to avoid fetching too many posts at once, and delta fetch to retrieve only relevant updates. Images and videos would be stored separately from post metadata and delivered through a CDN. On mobile, I would use local caching, lazy loading, placeholders, and suitable media resolutions. Finally, I would monitor latency, cache hit rates, and database performance to identify bottlenecks.

**Core idea:** Fetch only the data you need, keep frequently accessed information close to where it is used, and avoid repeatedly transferring large files.
