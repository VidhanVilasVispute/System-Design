# Stage 3 — Topic 8: Object Storage

## Theory

Every application eventually needs to store files — product images, user avatars, invoices, video recordings, backup archives, log files. The question is where and how.

Object storage is the answer for large, unstructured binary data at scale. It is the storage model behind AWS S3, Google Cloud Storage, and Azure Blob Storage — the backbone of how the modern internet stores files.

**The three storage models to distinguish upfront:**

```
Block Storage:
  Raw storage split into fixed-size blocks
  OS sees it as a hard drive — formats it with a filesystem
  Attached to one server at a time (usually)
  e.g. AWS EBS, local SSD
  Use: OS disks, database storage, VM disks

File Storage:
  Hierarchical directory/file structure
  Shared across multiple servers via network (NFS, NAS)
  e.g. AWS EFS, NFS
  Use: shared configuration files, legacy apps expecting filesystem

Object Storage:
  Flat namespace — no directories, no hierarchy
  Each object: unique key + data + metadata
  Accessed via HTTP API — not mounted as filesystem
  e.g. AWS S3, Google Cloud Storage, MinIO
  Use: images, videos, backups, archives, logs, any large binary data
  
The critical difference:
  Block/File: mutate files in place, seek to offset, append
  Object:     write once, replace entirely, no partial updates
              immutable after write — read the whole object or nothing
```

---

## Object Storage Internals

### What an Object Is

```
Object = Key + Data + Metadata

Key:
  Flat string identifier — the "path"
  "products/images/p-456/thumbnail.webp"
  "invoices/2026/04/inv-789.pdf"
  "backups/2026-04-08/db-snapshot.tar.gz"
  
  No real directory hierarchy — just a string with slashes
  The "/" is part of the key name — not a real directory separator
  Bucket can hold billions of objects with similar key prefixes

Data:
  Raw bytes — any format, any size
  Typically 1KB to 5TB per object (S3 max: 5TB)
  Stored redundantly across multiple physical locations

Metadata:
  System metadata:  content-type, content-length, etag, last-modified
  User metadata:    custom key-value pairs you attach
                    "x-amz-meta-uploader: u-123"
                    "x-amz-meta-original-filename: nike-shoes.jpg"
  
  Metadata is indexed — searchable
  Data is not indexed — cannot query by content
```

### How S3 Stores Objects Internally

```
Bucket: shopsphere-images
Object: products/p-456/hero.webp

1. Object split into parts (for large files):
   Part 1: bytes 0        to 5,242,879    (5MB)
   Part 2: bytes 5,242,880 to 10,485,759  (5MB)
   Part 3: bytes 10,485,760 to 12,000,000 (remaining)

2. Parts stored across multiple physical disks and AZs:
   Part 1 → AZ-a disk 3, AZ-b disk 7, AZ-c disk 1
   Part 2 → AZ-a disk 9, AZ-b disk 2, AZ-c disk 5
   Part 3 → AZ-a disk 1, AZ-b disk 4, AZ-c disk 8

3. Erasure coding instead of full replication:
   Object split into N data shards + K parity shards
   Can reconstruct from any N shards — tolerates K failures
   More storage-efficient than 3× replication
   S3 stores data across 3+ AZs — 99.999999999% (11 nines) durability

4. Metadata stored in separate distributed index:
   Key → object location mapping
   Allows O(1) object lookup by key
```

### Consistency Model

```
S3 consistency model (as of December 2020):
  Strong read-after-write consistency for all operations
  
  PUT object  → immediately visible on subsequent GET
  DELETE object → immediately invisible on subsequent GET
  LIST objects → immediately reflects latest state
  
  Before 2020: eventual consistency for overwrite PUTs
  Now: always strongly consistent — simpler to reason about
  
  Note: consistency is per-key, not transactional across keys
  Two objects cannot be updated atomically in S3
```

---

## S3 API — The Core Operations

```java
// Spring Boot integration with AWS S3 via SDK v2
@Service
public class S3StorageService {

    @Autowired private S3Client s3Client;
    private static final String BUCKET = "shopsphere-uploads";

    // ── UPLOAD ─────────────────────────────────────────────────────

    // Simple upload — for small files (< 100MB)
    public String uploadFile(String key, byte[] data, String contentType) {
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(BUCKET)
            .key(key)
            .contentType(contentType)
            .contentLength((long) data.length)
            .metadata(Map.of(
                "uploaded-by", "shopsphere-api",
                "environment", "production"
            ))
            .build();

        s3Client.putObject(request, RequestBody.fromBytes(data));
        return "https://shopsphere-uploads.s3.ap-south-1.amazonaws.com/" + key;
    }

    // Multipart upload — for large files (> 100MB)
    public String uploadLargeFile(String key, InputStream inputStream,
                                   long fileSize, String contentType) {

        // 1. Initiate multipart upload
        CreateMultipartUploadResponse initResponse = s3Client
            .createMultipartUpload(b -> b
                .bucket(BUCKET)
                .key(key)
                .contentType(contentType));

        String uploadId = initResponse.uploadId();
        List<CompletedPart> completedParts = new ArrayList<>();
        int partNumber = 1;
        int partSize = 10 * 1024 * 1024; // 10MB parts

        try {
            byte[] buffer = new byte[partSize];
            int bytesRead;

            while ((bytesRead = inputStream.read(buffer)) > 0) {
                byte[] partData = Arrays.copyOf(buffer, bytesRead);
                int currentPart = partNumber;

                // 2. Upload each part
                UploadPartResponse partResponse = s3Client.uploadPart(
                    b -> b.bucket(BUCKET)
                          .key(key)
                          .uploadId(uploadId)
                          .partNumber(currentPart),
                    RequestBody.fromBytes(partData)
                );

                completedParts.add(CompletedPart.builder()
                    .partNumber(partNumber)
                    .eTag(partResponse.eTag())
                    .build());

                partNumber++;
            }

            // 3. Complete multipart upload
            s3Client.completeMultipartUpload(b -> b
                .bucket(BUCKET)
                .key(key)
                .uploadId(uploadId)
                .multipartUpload(m -> m.parts(completedParts)));

            return buildObjectUrl(key);

        } catch (Exception e) {
            // Abort on failure — prevents orphaned parts (you pay for storage)
            s3Client.abortMultipartUpload(b -> b
                .bucket(BUCKET).key(key).uploadId(uploadId));
            throw new StorageException("Multipart upload failed", e);
        }
    }

    // ── DOWNLOAD ───────────────────────────────────────────────────

    public byte[] downloadFile(String key) {
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(BUCKET)
            .key(key)
            .build();

        return s3Client.getObjectAsBytes(request).asByteArray();
    }

    // Streaming download — for large files
    public void streamToResponse(String key, HttpServletResponse response)
            throws IOException {
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(BUCKET).key(key).build();

        try (ResponseInputStream<GetObjectResponse> s3Object =
                 s3Client.getObject(request)) {

            GetObjectResponse metadata = s3Object.response();
            response.setContentType(metadata.contentType());
            response.setContentLengthLong(metadata.contentLength());

            // Stream in chunks — never load entire file into memory
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = s3Object.read(buffer)) != -1) {
                response.getOutputStream().write(buffer, 0, bytesRead);
            }
        }
    }

    // ── DELETE ─────────────────────────────────────────────────────

    public void deleteFile(String key) {
        s3Client.deleteObject(b -> b.bucket(BUCKET).key(key));
    }

    // Batch delete — up to 1000 objects per request
    public void deleteFiles(List<String> keys) {
        List<ObjectIdentifier> objectIds = keys.stream()
            .map(k -> ObjectIdentifier.builder().key(k).build())
            .collect(Collectors.toList());

        s3Client.deleteObjects(b -> b
            .bucket(BUCKET)
            .delete(d -> d.objects(objectIds)));
    }

    // ── LIST ───────────────────────────────────────────────────────

    public List<String> listObjects(String prefix) {
        ListObjectsV2Request request = ListObjectsV2Request.builder()
            .bucket(BUCKET)
            .prefix(prefix)             // e.g. "products/p-456/"
            .maxKeys(1000)
            .build();

        return s3Client.listObjectsV2Paginator(request)
            .stream()
            .flatMap(response -> response.contents().stream())
            .map(S3Object::key)
            .collect(Collectors.toList());
    }
}
```

---

## Pre-Signed URLs — Secure Direct Access

One of the most important S3 patterns. Instead of proxying files through your application server, let clients upload/download directly to/from S3:

```
Without pre-signed URLs (bad):
  Client → your server → S3
  
  Problems:
    Your server becomes the bottleneck for every file
    Large file uploads consume your server's bandwidth
    Scaling file serving requires scaling application servers
    Video streaming through your server = expensive

With pre-signed URLs (correct):
  Upload: Client → S3 directly (server just generates the URL)
  Download: Client → S3/CDN directly (no server involvement)
  
  Benefits:
    Your server only generates URLs — minimal load
    S3 handles all bandwidth
    Client uploads directly — no double-transfer
    Works with CDN — serve files from edge
```

```java
@Service
public class PresignedUrlService {

    @Autowired private S3Presigner presigner;
    private static final String BUCKET = "shopsphere-uploads";

    // Generate pre-signed UPLOAD URL
    // Client uploads directly to S3 without touching your server
    public PresignedUploadUrl generateUploadUrl(
            String userId, String fileName, String contentType) {

        // Validate content type — only allow images
        if (!contentType.startsWith("image/")) {
            throw new InvalidFileTypeException("Only images allowed");
        }

        // Generate unique key — prevent path traversal and collisions
        String key = "products/images/" + userId + "/"
            + UUID.randomUUID() + "/"
            + sanitizeFileName(fileName);

        PutObjectRequest objectRequest = PutObjectRequest.builder()
            .bucket(BUCKET)
            .key(key)
            .contentType(contentType)
            // Content-Length condition — prevent huge uploads
            .build();

        PresignedPutObjectRequest presignedRequest = presigner
            .presignPutObject(r -> r
                .signatureDuration(Duration.ofMinutes(15)) // URL expires in 15 min
                .putObjectRequest(objectRequest));

        return PresignedUploadUrl.builder()
            .uploadUrl(presignedRequest.url().toString())
            .objectKey(key)
            .expiresAt(Instant.now().plus(Duration.ofMinutes(15)))
            .build();
    }

    // Generate pre-signed DOWNLOAD URL
    // For private content that should not be publicly accessible
    public String generateDownloadUrl(String key, Duration validity) {
        GetObjectRequest objectRequest = GetObjectRequest.builder()
            .bucket(BUCKET)
            .key(key)
            .build();

        PresignedGetObjectRequest presignedRequest = presigner
            .presignGetObject(r -> r
                .signatureDuration(validity)
                .getObjectRequest(objectRequest));

        return presignedRequest.url().toString();
    }
}

// Upload flow — frontend and backend coordination
@RestController
public class FileUploadController {

    @Autowired private PresignedUrlService presignedUrlService;
    @Autowired private ProductService productService;

    // Step 1: Client requests upload URL
    @PostMapping("/api/v1/products/{productId}/images/upload-url")
    public ResponseEntity<PresignedUploadUrl> getUploadUrl(
            @PathVariable String productId,
            @RequestParam String fileName,
            @RequestParam String contentType,
            @AuthenticationPrincipal UserDetails user) {

        PresignedUploadUrl uploadUrl = presignedUrlService
            .generateUploadUrl(user.getId(), fileName, contentType);

        return ResponseEntity.ok(uploadUrl);
    }

    // Step 2: Client uploads directly to S3 using the pre-signed URL
    // (This happens on the client — no server involvement)

    // Step 3: Client notifies server that upload is complete
    @PostMapping("/api/v1/products/{productId}/images/confirm")
    public ResponseEntity<ProductImage> confirmUpload(
            @PathVariable String productId,
            @RequestBody ConfirmUploadRequest request) {

        // Verify object actually exists in S3 before saving to DB
        s3Client.headObject(b -> b
            .bucket("shopsphere-uploads")
            .key(request.getObjectKey()));

        // Save image reference to database
        ProductImage image = productService.addImage(
            productId, request.getObjectKey());

        return ResponseEntity.status(201).body(image);
    }
}
```

**Client-side upload with pre-signed URL:**
```javascript
// Frontend JavaScript
async function uploadProductImage(file, productId) {

    // Step 1: Get pre-signed URL from your server
    const { uploadUrl, objectKey } = await fetch(
        `/api/v1/products/${productId}/images/upload-url?` +
        `fileName=${file.name}&contentType=${file.type}`
    ).then(r => r.json());

    // Step 2: Upload DIRECTLY to S3 — no server involved
    await fetch(uploadUrl, {
        method: 'PUT',
        body: file,
        headers: { 'Content-Type': file.type }
    });

    // Step 3: Confirm upload with your server
    await fetch(`/api/v1/products/${productId}/images/confirm`, {
        method: 'POST',
        body: JSON.stringify({ objectKey })
    });
}
```

---

## S3 Bucket Policies and Access Control

```
Access levels:
  Private (default):    Only bucket owner can access
  Public-read:          Anyone can read — for CDN-served assets
  Pre-signed URL:       Time-limited access for specific objects
  IAM Role:             AWS services (EC2, Lambda) access without credentials

Bucket policy for ShopSphere:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForProductImages",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::shopsphere-images/products/*"
    },
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::shopsphere-uploads",
        "arn:aws:s3:::shopsphere-uploads/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
```

---

## S3 Storage Classes — Cost Optimisation

S3 offers multiple storage tiers with different cost/access trade-offs:

```
S3 Standard:
  Cost:        High (~$0.023/GB/month)
  Availability: 99.99%
  Retrieval:   Milliseconds
  Use:         Frequently accessed data — active product images, recent uploads

S3 Standard-IA (Infrequent Access):
  Cost:        Lower (~$0.0125/GB/month) + retrieval fee
  Availability: 99.9%
  Retrieval:   Milliseconds
  Use:         Older product images, less popular content
               Break-even at < 1 access per month

S3 Glacier Instant Retrieval:
  Cost:        Very low (~$0.004/GB/month) + retrieval fee
  Retrieval:   Milliseconds
  Use:         Archive that still needs occasional fast access
               Old invoices, historical order data

S3 Glacier Flexible Retrieval:
  Cost:        Very low (~$0.0036/GB/month)
  Retrieval:   Minutes to hours
  Use:         Disaster recovery backups, compliance archives
               Data retrieved < once per year

S3 Glacier Deep Archive:
  Cost:        Cheapest (~$0.00099/GB/month)
  Retrieval:   Hours (12-48 hours)
  Use:         7-year compliance archives, legal holds
               Regulatory requirements to retain but never access
```

**Lifecycle policies — automatic tiering:**
```json
{
  "Rules": [
    {
      "Id": "ShopSphereImageLifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "products/images/" },
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER_IR"
        }
      ]
    },
    {
      "Id": "InvoiceArchival",
      "Status": "Enabled",
      "Filter": { "Prefix": "invoices/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555  // Delete after 7 years (regulatory compliance)
      }
    }
  ]
}
```

---

## S3 Versioning — Protecting Against Accidents

```
Versioning enabled on bucket:
  Every overwrite creates a new version — old version preserved
  Every delete creates a delete marker — object not actually deleted
  
  Bucket: shopsphere-uploads
  Key: products/p-456/hero.webp
  
  Version history:
    v3 (current): uploaded 2026-04-08  ← latest
    v2:           uploaded 2026-04-01
    v1:           uploaded 2026-03-15  ← oldest
  
  Restore v1:
    GET /products/p-456/hero.webp?versionId=<v1-id>
    
  Accidental delete:
    Delete creates delete marker — object appears gone
    Delete the delete marker → object restored
    True deletion requires deleting specific version IDs

Cost: pay for all versions — lifecycle policy to expire old versions
```

---

## S3 Event Notifications — Triggering Workflows

S3 can publish events when objects are created, deleted, or restored — enabling event-driven processing:

```
Product image upload flow with S3 events:

1. Seller uploads raw product photo to S3
   → PUT products/uploads/raw/p-456/photo.jpg

2. S3 publishes ObjectCreated event to SQS/SNS/Lambda:
   {
     "eventName": "ObjectCreated:Put",
     "s3": {
       "bucket": { "name": "shopsphere-uploads" },
       "object": {
         "key": "products/uploads/raw/p-456/photo.jpg",
         "size": 4500000
       }
     }
   }

3. Lambda function (or Spring Boot worker) triggered:
   - Download raw image from S3
   - Resize to multiple dimensions:
       hero:      1200×800px
       thumbnail: 200×200px
       mobile:    600×400px
   - Convert to WebP format (smaller, faster)
   - Upload processed images to S3:
       products/images/p-456/hero.webp
       products/images/p-456/thumbnail.webp
       products/images/p-456/mobile.webp
   - Update product record in DB with new image URLs
   - Invalidate CDN cache for this product

4. Processed images served directly from CDN
```

```java
// Spring Boot S3 event consumer (via SQS)
@SqsListener("image-processing-queue")
public void processImageUpload(S3EventNotification.S3EventNotificationRecord record) {
    String bucket = record.getS3().getBucket().getName();
    String key = record.getS3().getObject().getKey();

    log.info("Processing new upload: {}", key);

    try {
        // Download raw image
        byte[] rawImage = storageService.downloadFile(key);

        // Process — resize and convert
        Map<String, byte[]> processed = imageProcessor.processProductImage(rawImage);

        // Upload processed versions
        String productId = extractProductId(key);
        processed.forEach((variant, imageBytes) -> {
            String processedKey = "products/images/" + productId +
                "/" + variant + ".webp";
            storageService.uploadFile(processedKey, imageBytes, "image/webp");
        });

        // Update DB
        productService.updateImages(productId, buildImageUrls(productId));

        // Invalidate CDN
        cdnService.invalidate("images.shopsphere.com/products/" + productId + "/*");

        // Delete raw upload (no longer needed)
        storageService.deleteFile(key);

    } catch (Exception e) {
        log.error("Image processing failed for key: {}", key, e);
        throw e; // SQS will retry
    }
}
```

---

## Self-Hosted Object Storage — MinIO

In development and for teams that cannot use cloud providers, **MinIO** is the S3-compatible open-source alternative:

```yaml
# Docker Compose — MinIO for local development
services:
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"    # API port
      - "9001:9001"    # Console UI
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

volumes:
  minio_data:
```

```java
// MinIO uses S3-compatible API — same SDK, different endpoint
@Bean
public S3Client s3Client() {
    if (environment.equals("development")) {
        return S3Client.builder()
            .endpointOverride(URI.create("http://localhost:9000"))
            .credentialsProvider(StaticCredentialsProvider.create(
                AwsBasicCredentials.create("minioadmin", "minioadmin")))
            .region(Region.US_EAST_1)
            .forcePathStyle(true)  // Required for MinIO
            .build();
    }
    // Production: uses IAM role, default AWS endpoint
    return S3Client.builder()
        .region(Region.AP_SOUTH_1)
        .build();
}
```

---

## Real-World Example — ShopSphere Complete Storage Architecture

```
ShopSphere S3 Bucket Structure:

shopsphere-uploads (private — pre-signed URL access):
  products/uploads/raw/         ← seller raw uploads (temporary)
  invoices/                     ← order invoices (private)
  exports/                      ← data export files (temporary)

shopsphere-images (public-read + CDN):
  products/images/{productId}/
    hero.webp
    thumbnail.webp
    mobile.webp
    gallery/1.webp, 2.webp...
  users/avatars/{userId}/avatar.webp
  banners/homepage/banner-1.webp

shopsphere-videos (private + CDN with signed URLs):
  products/videos/{productId}/
    demo.m3u8           ← HLS manifest
    segment_001_1080p.ts
    segment_001_720p.ts
    ...

shopsphere-backups (versioned, lifecycle to Glacier):
  daily/2026-04-08/db-snapshot.tar.gz
  weekly/2026-04-06/full-backup.tar.gz

shopsphere-logs (lifecycle to Glacier after 30 days):
  application/2026/04/08/api-gateway.log.gz
  access/2026/04/08/nginx-access.log.gz
```

**Cost optimisation results:**
```
Without lifecycle policies:
  All objects in S3 Standard
  Monthly bill: $4,200

With lifecycle policies:
  Active images:    Standard    (0-90 days)
  Older images:     Standard-IA (90+ days)
  Historical data:  Glacier     (365+ days)
  Monthly bill: $890

Savings: 79% cost reduction
```

---

## Interview Q&A

**Q: What is object storage and how does it differ from block and file storage?**
Block storage presents raw storage as a device the OS can format and mount — databases and operating systems use it because they need to read and write at specific byte offsets. File storage presents a hierarchical directory structure accessible over a network — useful for shared configuration or legacy applications. Object storage is a flat namespace of key-value pairs accessible via HTTP API — each object is read or written as a complete unit, with no partial updates or random access. Object storage scales to petabytes with eleven nines of durability because data is distributed and erasure-coded across multiple availability zones. It is the correct choice for images, videos, backups, and any large binary content.

**Q: What is a pre-signed URL and why should file uploads go directly to S3?**
A pre-signed URL is a time-limited, cryptographically signed URL that grants temporary access to a specific S3 object for upload or download without requiring AWS credentials. For file uploads, the client requests a pre-signed URL from your server, then uploads the file directly to S3 using that URL — your application server is never in the data path. This eliminates your server as the bandwidth bottleneck for large files, prevents you from paying double bandwidth costs by proxying through your server, and lets S3 handle the scale of concurrent uploads without affecting your application's performance.

**Q: How would you design an image processing pipeline for ShopSphere product photos?**
The seller uploads the raw image to S3 with a pre-signed URL. S3 publishes an ObjectCreated event to SQS. A worker consumes from SQS, downloads the raw image, resizes it to multiple variants — hero, thumbnail, mobile — and converts to WebP format for smaller file size. Each variant is uploaded to a separate S3 path. The product record in the database is updated with the new image URLs, and the CDN cache is invalidated for that product. The original raw upload is deleted. This pipeline is fully asynchronous — the seller gets immediate confirmation of upload, and image processing happens in the background without blocking the API.

**Q: What are S3 storage classes and when would you use them?**
S3 Standard is for frequently accessed data — active product images and recent uploads where sub-millisecond retrieval matters and cost is secondary. Standard-IA is for infrequently accessed data — images for products that rarely get viewed, where retrieval is still millisecond-fast but you pay a per-retrieval fee making it cost-effective for data accessed less than once a month. Glacier Instant Retrieval is for archive data that still needs occasional fast access — old invoices, historical records. Glacier Deep Archive is for regulatory compliance data that must be retained but almost never accessed — seven-year audit archives with 12 to 48 hour retrieval times at a fraction of the cost. Lifecycle policies automate transitions between classes based on object age.

**Q: How does S3 achieve eleven nines of durability?**
S3 achieves 99.999999999% durability by distributing every object across a minimum of three availability zones within a region using erasure coding. Erasure coding splits each object into data shards and parity shards — the object can be fully reconstructed from any subset of shards, tolerating multiple simultaneous failures. This is more storage-efficient than full triplication while providing stronger durability guarantees. AWS continuously monitors for bit rot, disk failures, and network errors, automatically detecting and repairing corrupted data by reconstructing it from intact shards. The probability of losing data stored in S3 Standard is approximately one object per hundred million objects per ten thousand years.

---

Say **"next"** when ready for Topic 9 — Different Storage Systems (the complete taxonomy).
