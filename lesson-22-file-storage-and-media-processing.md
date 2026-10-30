# Lesson 22 — File Storage and Media Processing: Object Storage, Uploads, CDNs, and Image/Video Pipelines

Almost every modern application handles files: profile pictures, documents, invoices, images, videos, and attachments.

As the application grows, storing files directly on application servers becomes difficult to scale. We need a dedicated storage architecture that handles uploads, downloads, processing, security, and delivery efficiently.

In this lesson, we'll explore object storage, file-upload architecture, pre-signed URLs, CDNs, media processing, and storage scalability.

## 1. Why File Storage Needs a Separate Architecture

Imagine our ride-booking platform supports:

- Passenger profile pictures.
- Driver profile pictures.
- Driver identity documents.
- Vehicle registration documents.
- Booking attachments.
- Invoice PDFs.
- Trip-related images.

Initially, we might store these files directly on the application server.

For example:

```text
Application Server
    |
    +---- /uploads/profile.jpg
    |
    +---- /uploads/document.pdf
    |
    +---- /uploads/invoice.pdf
```

This looks simple, but it introduces problems as the system grows.

### Problems with Local File Storage

**1. Multiple application servers**

Suppose we have three application servers:

```text
             Load Balancer
                  |
          +-------+-------+
          |       |       |
          v       v       v
        App 1   App 2   App 3
```

A file uploaded to App 1 may not exist on App 2.

If a later request reaches App 2, the file might not be available.

**2. Server replacement**

If an application server is replaced during deployment or autoscaling, locally stored files may disappear.

**3. Storage capacity**

Application servers are designed primarily to execute application logic, not to act as large-scale file-storage systems.

**4. Backup and replication**

We need separate strategies to protect files and make them available across infrastructure failures.

**5. Delivery performance**

Serving large files through application servers consumes application bandwidth and resources.

The solution is to separate file storage from application compute.

---

## 2. Types of Storage

There are three common storage models.

### 2.1 Block Storage

Block storage provides raw storage volumes divided into blocks.

Examples:
- AWS EBS.
- Azure Managed Disks.
- Google Persistent Disk.

Typical use cases:
- Database storage.
- Operating system disks.
- Persistent application volumes.

Think of block storage as a disk attached to a machine.

### 2.2 File Storage

File storage organizes data into files and directories.

Examples:
- NFS.
- Amazon EFS.
- Azure Files.

Typical use cases:
- Shared directories.
- Applications requiring a filesystem interface.
- Shared files across multiple servers.

Example:

```text
/shared
    |
    +---- profiles/
    |
    +---- documents/
    |
    +---- invoices/
```

Multiple application servers can access a shared filesystem, subject to its performance and availability characteristics.

### 2.3 Object Storage

Object storage stores data as objects inside containers called buckets.

Examples:
- Amazon S3.
- Google Cloud Storage.
- Azure Blob Storage.

An object generally consists of:

- Object data.
- Object key.
- Metadata.

Example:

```text
Bucket: ride-booking-files

Objects:
  profiles/passenger-123/avatar.jpg
  profiles/driver-456/avatar.jpg
  documents/driver-456/license.pdf
  invoices/booking-789.pdf
```

Object storage is commonly used for images, videos, documents, backups, and static assets.

### Comparison

| Feature | Block Storage | File Storage | Object Storage |
|---|---|---|---|
| Data organization | Blocks | Files and directories | Objects and keys |
| Access pattern | Disk I/O | Filesystem operations | API requests |
| Typical use | Databases | Shared files | Media and documents |
| Scaling | Volume-based | Filesystem-based | Designed for large-scale object storage |
| Metadata | Filesystem/application | File attributes | Object metadata |

**Key takeaway:** Object storage is generally the default choice for large-scale application file storage.

---

## 3. Object Storage Architecture

A typical architecture separates metadata from file contents.

```text
              Client
                |
                v
          API Gateway
                |
                v
         Application Server
                |
          +-----+------+
          |            |
          v            v
      PostgreSQL   Object Storage
      (Metadata)   (File Content)
```

### What Goes Where?

**PostgreSQL stores metadata:**

| id | owner_id | object_key | content_type | status |
|---|---|---|---|---|
| 101 | P123 | profiles/P123/avatar.jpg | image/jpeg | READY |
| 102 | D456 | documents/D456/license.pdf | application/pdf | READY |

**Object storage stores the actual file bytes.**

Why separate them?

- Database remains focused on transactional and relational data.
- Object storage handles large binary objects.
- Metadata can be queried efficiently.
- Files can be delivered independently through a CDN.

### Important Object Storage Concepts

**Bucket:** A logical container for objects.

**Object key:** A unique identifier within a bucket.

**Metadata:** Information about an object, such as content type, size, and custom attributes.

**Versioning:** Allows multiple versions of an object to be retained.

**Lifecycle policy:** Automatically transitions or deletes objects based on configured rules.

**Durability:** The ability to preserve data over time despite hardware failures.

Do not confuse durability with availability. A storage system can be highly durable without every object being instantly accessible.

---

## 4. Direct Upload vs Application-Mediated Upload

There are two common ways to upload files.

### 4.1 Application-Mediated Upload

The client sends the file to the backend, which uploads it to object storage.

```text
Client
  |
  | Upload file
  v
Application Server
  |
  | Upload file
  v
Object Storage
```

Advantages:
- Backend can inspect files before accepting them.
- Centralized upload handling.
- Application can enforce policies directly.

Disadvantages:
- Backend handles the entire file transfer.
- Increased bandwidth consumption.
- Application servers spend resources handling large payloads.
- Can become a bottleneck.

This can be suitable for small files or applications that need centralized streaming inspection.

### 4.2 Direct-to-Object-Storage Upload

The backend authorizes the upload and generates a pre-signed URL.

The client uploads directly to object storage.

```text
Client
  |
  | 1. Request upload permission
  v
Application Server
  |
  | 2. Generate pre-signed URL
  v
Client
  |
  | 3. Upload file directly
  v
Object Storage
  |
  | 4. Upload completed
  v
Application Server
```

Advantages:
- Application servers do not carry the file payload.
- Better scalability for large uploads.
- Reduced backend bandwidth.
- Object storage handles the transfer.

Disadvantages:
- More complex upload lifecycle.
- Requires careful validation and authorization.
- Application must verify that the upload completed successfully.

For large-scale media uploads, direct upload is often a better architecture.

---

## 5. Pre-Signed URLs

A pre-signed URL is a temporary URL that grants permission to perform a specific operation on an object.

For example:
- Upload an object.
- Download an object.

A pre-signed URL typically includes a signature and expiration information.

### Upload Flow

```text
1. Client
     |
     | POST /uploads
     v
2. Backend
     |
     | Authenticate user
     | Authorize upload
     | Generate unique object key
     | Generate pre-signed URL
     v
3. Client
     |
     | PUT file using pre-signed URL
     v
4. Object Storage
     |
     | Store file
     v
5. Client
     |
     | Notify backend of completion
     v
6. Backend
     |
     | Verify object
     | Update metadata status
     v
7. PostgreSQL
```

### Example API Design

**Step 1 — Request upload**

```http
POST /api/uploads
```

Request:

```json
{
  "fileName": "profile.jpg",
  "contentType": "image/jpeg",
  "size": 250000
}
```

Response:

```json
{
  "uploadId": "UP123",
  "uploadUrl": "https://storage.example.com/...",
  "objectKey": "profiles/P123/random-id.jpg",
  "expiresIn": 300
}
```

**Step 2 — Upload directly**

```http
PUT <pre-signed-url>
```

The client sends the file bytes directly to object storage.

**Step 3 — Confirm completion**

```http
POST /api/uploads/UP123/complete
```

The backend verifies the uploaded object before marking it as available.

### Pre-Signed URL Security

Remember:

1. URLs should expire quickly.
2. Scope permissions to a specific object and operation.
3. Do not expose cloud credentials to the client.
4. Generate unpredictable object keys.
5. Enforce maximum file sizes.
6. Validate the actual file content.
7. Verify ownership before allowing downloads.
8. Avoid logging full signed URLs because they contain temporary credentials.

A pre-signed URL is a bearer capability: anyone who obtains it may be able to use it within its permissions and lifetime.

---

## 6. File Upload Security

Accepting files introduces security risks.

A file extension alone does not prove that the file is safe.

For example:

```text
malicious-file.jpg
```

The filename might end in `.jpg`, but its content could be something else.

### Important Validations

**1. File size**

Set maximum sizes.

Example:

```text
Profile picture: 5 MB
PDF document: 20 MB
Video attachment: 500 MB
```

These are example limits, not universal standards.

**2. Content type**

Check the declared MIME type, but do not trust it alone.

**3. File signature**

Inspect file signatures or magic bytes to help identify the actual format.

**4. Malware scanning**

Scan uploaded files before making them available for sensitive use cases.

**5. Filename safety**

Do not use user-provided filenames directly as storage paths.

Generate unique object keys.

**6. Access control**

Private documents should not be publicly accessible.

**7. Storage isolation**

Store untrusted uploads in a quarantine area until validation is complete.

### Secure Upload Lifecycle

```text
Upload Requested
      |
      v
Temporary Upload
      |
      v
Validation
      |
      v
Malware Scan
      |
      v
Approved?
   /       \
 Yes       No
  |         |
  v         v
Available  Quarantine/Delete
```

For identity documents and other sensitive files, consider encryption, strict access controls, audit trails, and retention policies.

---

## 7. CDN for File Delivery

Suppose our ride-booking platform has 100,000 users distributed across different regions.

All profile images are stored in object storage in one region.

Without a CDN:

```text
User in India
     |
     v
Application / Storage Region
     |
     v
Object Storage
```

Users far away from the storage region may experience higher latency.

A Content Delivery Network (CDN) caches content at geographically distributed edge locations.

```text
                 User
                   |
                   v
                CDN Edge
                   |
            +------+------+
            |             |
         Cache Hit     Cache Miss
            |             |
            v             v
          Return       Origin Storage
          Content          |
                            v
                       Cache Content
```

### Cache Hit

The requested file is already cached at the edge.

The CDN returns it without contacting the origin.

### Cache Miss

The CDN fetches the file from the origin, returns it to the user, and may cache it.

### Benefits of CDN

- Lower latency.
- Reduced origin traffic.
- Better global delivery.
- Improved scalability.
- Protection features when combined with appropriate CDN security services.

### What Should We Cache?

| Content | Typical Strategy |
|---|---|
| Public profile images | Long TTL with versioned URLs |
| Static JavaScript/CSS | Long TTL with content hashes |
| Public product images | Long TTL |
| Private invoices | Private delivery with authorization |
| Sensitive identity documents | Avoid public caching |

A CDN does not automatically make private content safe.

---

## 8. Public vs Private File Access

Not every file should be publicly accessible.

### Public Files

Examples:
- Public profile pictures.
- Public marketing images.
- Static assets.

These can often be delivered through public CDN URLs.

### Private Files

Examples:
- Identity documents.
- Invoices.
- Private booking attachments.
- Medical or financial documents.

These require authorization.

A common download flow:

```text
Client
  |
  | Request document
  v
Application Server
  |
  | Authenticate user
  | Verify ownership / permissions
  v
Generate short-lived download URL
  |
  v
Client
  |
  | Download
  v
Private Object Storage
```

The application decides whether the user is allowed to download the file. The signed URL then grants temporary access.

For highly sensitive content, consider whether a proxy-based download or stricter edge authorization is more appropriate.

---

## 9. Image Processing Architecture

Uploading an image does not mean we should serve the original file for every use case.

Imagine a profile image uploaded at 4000 × 4000 pixels.

Our application may need:
- 64 × 64 thumbnail.
- 256 × 256 profile image.
- 1024 × 1024 larger image.

Generating these versions improves bandwidth usage and client performance.

### Image Processing Flow

```text
Client
  |
  v
Object Storage
  |
  | Upload event
  v
Message Queue
  |
  v
Image Processing Worker
  |
  +------> Thumbnail
  |
  +------> Medium Image
  |
  +------> Large Image
  |
  v
Object Storage
  |
  v
CDN
```

Why use asynchronous processing?

Image processing can consume significant CPU and memory.

If performed inside the upload request, it increases API response time and ties up application resources.

Instead:
1. Accept the upload.
2. Store the original.
3. Publish a processing event.
4. Return an upload status.
5. Process the image in the background.
6. Mark the image as ready when processing succeeds.

### Image Optimization

Common techniques:
- Resize.
- Compress.
- Convert formats.
- Strip unnecessary metadata.
- Generate responsive sizes.

Modern formats such as WebP and AVIF can reduce image size, depending on image content and quality settings.

---

## 10. Video Processing Architecture

Video processing is more complex because files are larger and require more computation.

A typical video pipeline:

```text
Client
  |
  v
Direct Upload
  |
  v
Object Storage (Original)
  |
  | Upload Event
  v
Processing Queue
  |
  v
Video Processing Workers
  |
  +------> 360p
  |
  +------> 720p
  |
  +------> 1080p
  |
  +------> Thumbnails
  |
  +------> HLS/DASH Segments
  |
  v
Processed Storage
  |
  v
CDN
  |
  v
Video Player
```

### Why Multiple Resolutions?

Different users have different:
- Network speeds.
- Device capabilities.
- Screen sizes.

Providing multiple resolutions enables adaptive bitrate streaming.

### HLS and DASH

Two common streaming protocols:

- HLS — HTTP Live Streaming.
- DASH — Dynamic Adaptive Streaming over HTTP.

Videos are divided into small segments. The player chooses an appropriate quality based on available bandwidth and playback conditions.

This is more flexible than downloading one large video file.

---

## 11. Asynchronous Processing and Event-Driven Design

Media processing is a good example of an asynchronous workflow.

Suppose 1,000 users upload images simultaneously.

If every request performs image processing synchronously, the API servers may become overloaded.

A queue separates upload traffic from processing capacity.

```text
Upload API
    |
    v
Object Storage
    |
    v
Message Broker
    |
    +------> Worker 1
    |
    +------> Worker 2
    |
    +------> Worker 3
```

### Important Design Considerations

**Queue depth:** How many jobs are waiting?

**Processing rate:** How many jobs are completed per second?

**Oldest job age:** How long has the oldest job been waiting?

**Retry strategy:** What happens when image processing fails?

**Dead-letter queue:** Where do repeatedly failing jobs go?

**Idempotency:** Can the same processing event be safely delivered more than once?

Workers should be idempotent because message delivery is commonly at least once.

For example, use a deterministic output key or a processing version so that duplicate events do not create inconsistent outputs.

---

## 12. Storage Capacity Planning

Let's estimate storage for our ride-booking platform.

Assume:

- 100,000 active drivers.
- Each driver uploads 2 documents.
- Each document averages 2 MB.

Total document storage:

```text
100,000 × 2 × 2 MB

= 400,000 MB

= 400 GB
```

Now assume 1 million passengers upload one profile picture each.

Average original image size: 1 MB.

```text
1,000,000 × 1 MB

= 1,000,000 MB

≈ 1 TB
```

Total original storage:

```text
400 GB + 1 TB = 1.4 TB
```

This is only a basic estimate.

A production estimate should also consider:
- Generated image variants.
- New uploads over time.
- Retention.
- Versioning.
- Replication and backup policies.
- Temporary processing files.
- Storage growth.
- Download and egress costs.

### Bandwidth Estimation

Suppose:
- 100,000 users view 20 profile images daily.
- Each delivered image averages 100 KB.

Daily delivery:

```text
100,000 × 20 × 100 KB

= 200,000,000 KB

≈ 200 GB/day
```

A CDN can reduce repeated origin downloads significantly when cacheability and cache hit rates are good.

---

## 13. Object Storage Lifecycle and Retention

Files should not necessarily remain in their original storage tier forever.

Lifecycle policies can automate storage management.

Example:

| File Age | Action |
|---|---|
| 0–30 days | Standard storage |
| 30–180 days | Infrequent-access tier |
| 180+ days | Archive tier, if appropriate |
| Retention expired | Delete, if legally and operationally permitted |

The correct policy depends on:
- Access frequency.
- Retrieval latency requirements.
- Regulatory obligations.
- Retention rules.
- Deletion requirements.

Be careful with archive storage: retrieval can take longer and may cost more.

### Deletion Is More Complex Than It Looks

Deleting a database row does not necessarily delete the associated object.

Likewise, deleting an object does not automatically remove its metadata.

Design a deletion workflow that handles:
- Database metadata.
- Original objects.
- Generated variants.
- CDN cache invalidation where necessary.
- Object versions.
- Backups and retention obligations.

For sensitive data, define and verify the complete deletion lifecycle.

---

## 14. Object Storage Reliability and Consistency

Object storage systems provide strong durability and availability characteristics, but application-level workflows still need failure handling.

Consider this sequence:

1. File uploads successfully.
2. Application crashes before updating PostgreSQL.
3. The object exists, but the database has no corresponding record.

This creates an orphaned object.

The reverse can also happen:

1. Database metadata is created.
2. Upload fails.
3. The database refers to an object that does not exist.

### Handling the Consistency Gap

Use an upload state machine:

```text
PENDING
   |
   v
UPLOADING
   |
   v
UPLOADED
   |
   v
PROCESSING
   |
   v
READY
```

Possible failure states:

```text
FAILED
REJECTED
EXPIRED
```

The backend should verify object existence and relevant metadata before marking an upload as ready.

Use:
- Expiration of incomplete uploads.
- Retryable completion workflows.
- Periodic reconciliation jobs.
- Cleanup of orphaned objects.
- Idempotent state transitions.

Do not assume a database transaction can atomically include an external object-storage operation.

---

## 15. Multipart Uploads

Large files can be uploaded in smaller parts.

For example, a 500 MB video might be divided into 10 MB chunks.

```text
500 MB File
    |
    +---- Part 1
    +---- Part 2
    +---- Part 3
    +---- ...
    +---- Part 50
```

The storage service assembles the parts after successful completion.

### Benefits

- Parallel uploads.
- Better handling of network interruptions.
- Retry only failed parts.
- More efficient large-file uploads.

### Important Considerations

- Track the multipart upload ID.
- Track uploaded part numbers and identifiers.
- Set an expiration or cleanup policy for incomplete uploads.
- Validate final size and object integrity.
- Complete the upload only after required parts are present.

Multipart upload is especially useful for large videos and other large media files.

---

## 16. Storage Security and Access Control

Object storage should follow least-privilege principles.

Important controls:

1. Keep private buckets private.
2. Use identity-based access policies.
3. Restrict permissions by bucket and object prefix.
4. Use short-lived signed URLs.
5. Encrypt stored objects.
6. Encrypt communication using TLS.
7. Enable audit logging where appropriate.
8. Avoid exposing storage credentials to clients.
9. Scan untrusted files.
10. Apply retention and deletion policies.

For example:

```text
Booking Service:
  Read/write booking attachments

Notification Service:
  Read approved notification assets

Passenger Client:
  No direct bucket credentials
  Temporary access only through authorized URLs
```

Different services should not automatically receive full access to all buckets.

---

## 17. Common File Storage Mistakes

1. Storing uploaded files on application-server disks.
2. Sending large file payloads through application servers unnecessarily.
3. Making every uploaded file publicly accessible.
4. Trusting file extensions or declared MIME types.
5. Using original filenames as object keys.
6. Forgetting to clean up incomplete multipart uploads.
7. Processing large videos synchronously.
8. Serving original high-resolution images everywhere.
9. Forgetting CDN cache policies for private content.
10. Not tracking processing status.
11. Assuming database and object storage operations are one transaction.
12. Ignoring storage egress costs.
13. Forgetting object versions and backup retention during deletion.
14. Not making processing jobs idempotent.

---

## 18. Revision Questions

1. What is the difference between block, file, and object storage?
2. Why is local file storage difficult to scale across multiple application servers?
3. What is object storage?
4. What information is stored in PostgreSQL versus object storage?
5. What is a pre-signed URL?
6. Explain direct upload versus application-mediated upload.
7. Why should pre-signed URLs expire?
8. How do you protect uploaded files from malicious content?
9. What is the difference between public and private file delivery?
10. How does a CDN improve file delivery?
11. Why should image processing happen asynchronously?
12. What is adaptive bitrate streaming?
13. Explain HLS and DASH.
14. Why must media-processing consumers be idempotent?
15. How would you estimate storage capacity?
16. What is multipart upload?
17. How do you handle orphaned objects?
18. How would you implement file retention and deletion?
19. What metrics would you monitor for a media-processing pipeline?
20. How would you protect private driver identity documents?

---

## 19. Practical Exercise — Design a File Storage System

Extend our ride-booking platform to support:

- Passenger profile pictures.
- Driver profile pictures.
- Driver identity documents.
- Booking invoices.
- Trip attachments.
- Video attachments.

Design an architecture that answers:

1. Which storage type would you choose and why?
2. Where would you store file metadata?
3. Would you use direct upload or application-mediated upload?
4. How would you generate and validate pre-signed URLs?
5. How would you prevent unauthorized access to identity documents?
6. How would you process images asynchronously?
7. How would you deliver files globally?
8. How would you handle a failed upload?
9. How would you prevent duplicate processing jobs?
10. How would you estimate storage and bandwidth requirements?
11. How would you handle file deletion and retention?
12. What dashboards and alerts would you create?

### Final Takeaway

A scalable file-storage architecture separates responsibilities:

- **Object Storage:** Stores original files and generated assets.
- **PostgreSQL:** Stores metadata and file lifecycle state.
- **Message Broker:** Coordinates asynchronous processing.
- **Workers:** Resize, compress, scan, and transcode files.
- **CDN:** Delivers cacheable content efficiently.
- **Application Services:** Enforce authentication, authorization, and upload policies.

The goal is to keep file transfer and media processing from becoming bottlenecks in the core application.

### Practice focus

Design the upload flow for a driver's identity document. Pay particular attention to private access, pre-signed URLs, validation, processing states, and deletion.