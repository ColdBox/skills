---
name: s3sdk
description: >
  Use this skill when working with Amazon S3 (or S3-compatible storage) in ColdBox/BoxLang using
  the s3sdk module. Covers injection, bucket operations, object upload/download/delete, presigned
  URLs, metadata, ACLs, multipart uploads, and integration with cbfs or direct usage.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# S3SDK Skill

## When to Use This Skill

Load this skill when:
- Uploading files to Amazon S3 or S3-compatible storage (MinIO, DigitalOcean Spaces, Backblaze)
- Generating presigned URLs for time-limited public or private file access
- Listing, copying, moving, or deleting S3 objects or buckets
- Setting object metadata, ACLs, or storage class
- Implementing large-file multipart uploads

## Installation

```bash
box install s3sdk
```

## Configuration

### config/modules/s3sdk.cfc

```js
function configure() {
    return {
        accessKey      : getSystemSetting( "S3_ACCESS_KEY_ID",     "" ),
        secretKey      : getSystemSetting( "S3_SECRET_ACCESS_KEY", "" ),
        region         : getSystemSetting( "S3_REGION",            "us-east-1" ),
        // For S3-compatible providers set a custom host:
        awsDomain      : getSystemSetting( "S3_ENDPOINT", "s3.amazonaws.com" ),
        ssl            : true,
        defaultBucket  : getSystemSetting( "S3_BUCKET", "my-app-bucket" ),
        signatureType  : "v4",
        throwOnError   : true
    }
}
```

## Core API

### Injection

```js
property name="s3" inject="AmazonS3@s3sdk";
```

### Object Operations

```js
// Upload from a file path
s3.putObject(
    bucketName   = "my-bucket",
    uri          = "uploads/profile-#userId#.jpg",
    filePath     = expandPath( "/tmp/upload.jpg" ),
    contentType  = "image/jpeg",
    acl          = "private"
)

// Upload binary/string content
s3.putObject(
    bucketName  = "my-bucket",
    uri         = "reports/#reportId#.json",
    data        = serializeJSON( reportData ),
    contentType = "application/json"
)

// Download to file
s3.getObjectSave(
    bucketName = "my-bucket",
    uri        = "uploads/document.pdf",
    destFile   = expandPath( "/tmp/document.pdf" )
)

// Get as binary
var bytes = s3.getObject( bucketName = "my-bucket", uri = "uploads/file.png" )

// Check existence
var exists = s3.objectExists( bucketName = "my-bucket", uri = "uploads/file.png" )

// Delete
s3.deleteObject( bucketName = "my-bucket", uri = "uploads/old-file.png" )

// Copy
s3.copyObject(
    fromBucket = "my-bucket",
    fromURI    = "uploads/original.jpg",
    toBucket   = "my-bucket",
    toURI      = "processed/thumb.jpg"
)

// List objects
var objects = s3.getBucket( bucketName = "my-bucket", prefix = "uploads/" )
```

### Presigned URLs

```js
// Presigned GET URL — expires in 15 minutes
var url = s3.getAuthenticatedURL(
    bucketName = "my-bucket",
    uri        = "private/document-#rc.id#.pdf",
    minutesValid = 15,
    method       = "GET"
)

// Presigned PUT URL — for direct browser uploads
var putUrl = s3.getAuthenticatedURL(
    bucketName   = "my-bucket",
    uri          = "uploads/user-#userId#/#createUUID()#.jpg",
    minutesValid = 5,
    method       = "PUT"
)
```

### Metadata

```js
var meta = s3.getObjectInfo(
    bucketName = "my-bucket",
    uri        = "uploads/file.pdf"
)
// meta.content-type, meta.content-length, meta.last-modified, meta.etag

s3.putObject(
    bucketName  = "my-bucket",
    uri         = "uploads/doc.pdf",
    filePath    = filePath,
    metaHeaders = {
        "x-amz-meta-uploaded-by" : userId,
        "x-amz-meta-original-name" : originalFileName
    }
)
```

## Production Patterns

### File Upload Handler

```js
function upload( event, rc, prc ) {
    var uploadedFile = event.getHTTPContent()
    var mimeType     = event.getHTTPHeader( "Content-Type" )
    var allowedTypes = [ "image/jpeg", "image/png", "application/pdf" ]

    // Validate MIME type
    if ( !allowedTypes.find( mimeType ) ) {
        return event.renderData(
            type       = "json",
            statusCode = 415,
            data       = { error: "Unsupported file type" }
        )
    }

    var key = "uploads/#security.getCurrentUser().getId()#/#createUUID()#"
    // Append safe extension based on MIME
    key &= ( mimeType == "application/pdf" ? ".pdf" : ".jpg" )

    s3.putObject(
        bucketName  = "my-bucket",
        uri         = key,
        data        = uploadedFile,
        contentType = mimeType,
        acl         = "private"
    )

    event.renderData( type = "json", statusCode = 201, data = {
        key : key,
        url : s3.getAuthenticatedURL( "my-bucket", key, 60 )
    } )
}
```

### Serve Private File via Presigned URL

```js
function download( event, rc, prc ) {
    var file = fileService.getOrFail( rc.id )

    // Verify user can access this file
    if ( file.getUserId() != security.getCurrentUser().getId() ) {
        security.secure( "admin" )
    }

    var url = s3.getAuthenticatedURL(
        bucketName   = "my-bucket",
        uri          = file.getS3Key(),
        minutesValid = 5,
        method       = "GET"
    )

    relocate( url )
}
```

### Cleanup Old Files

```js
function purgeOldUploads() {
    var cutoff  = dateAdd( "d", -30, now() )
    var objects = s3.getBucket( bucketName = "my-bucket", prefix = "temp/" )

    for ( var obj in objects ) {
        if ( obj.lastModified < cutoff ) {
            s3.deleteObject( bucketName = "my-bucket", uri = obj.key )
        }
    }
}
```

## Best Practices

- **Store credentials in environment variables** — never hardcode access keys in source
- **Use `acl = "private"` for user uploads** — only generate presigned URLs for access
- **Validate MIME type server-side** before accepting uploads — do not trust client-provided Content-Type
- **Generate a random key/UUID for each upload** — never use user-supplied filenames directly (path traversal risk)
- **Set short expiry on presigned GET URLs** (≤15 min for sensitive files)
- **Use presigned PUT URLs for direct browser uploads** — avoids routing large files through your server
- **Namespace keys by user ID** (`uploads/{userId}/{uuid}`) to enable easy per-user cleanup
- **Enable S3 server-side encryption** (`x-amz-server-side-encryption: AES256`) for sensitive data

## Documentation

- s3sdk: https://github.com/coldbox-modules/s3sdk
- Amazon S3 API: https://docs.aws.amazon.com/AmazonS3/latest/API/
