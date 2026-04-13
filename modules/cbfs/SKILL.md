---
name: cbfs
description: >
  Use this skill when working with the cbfs file system abstraction module in ColdBox/BoxLang applications.
  Covers installation, disk configuration (local, S3, RAM, FTP), CRUD file operations, streaming, URL
  generation, file uploads, directory listing, and multi-provider patterns for different environments.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBFS Skill

## When to Use This Skill

Load this skill when:
- Reading, writing, copying, moving, or deleting files across storage providers
- Configuring local disk for development and S3/cloud disk in production
- Handling file uploads and storing them to a named disk
- Generating public or signed URLs to stored files
- Listing directory contents or checking file existence
- Abstracting storage so the same code works locally and in the cloud

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install cbfs
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbfs = {
        defaultDisk = "local",
        disks = {
            local = {
                provider   = "Local@cbfs",
                properties = {
                    path = expandPath( "/storage" )
                }
            },
            s3 = {
                provider   = "S3@cbfs",
                properties = {
                    accessKey = getSystemSetting( "AWS_ACCESS_KEY" ),
                    secretKey = getSystemSetting( "AWS_SECRET_KEY" ),
                    bucket    = getSystemSetting( "S3_BUCKET", "my-app-files" ),
                    region    = getSystemSetting( "AWS_REGION",  "us-east-1" )
                }
            },
            ram = {
                provider   = "RAM@cbfs",
                properties = {}
            }
        }
    }
}
```

> Store AWS credentials in environment variables — never hardcode them.

## Core API

### Injection

```js
// Inject the DiskService; then call .disk() to select a provider
property name="diskService" inject="DiskService@cbfs";

// Or inject a specific disk directly
property name="localDisk" inject="disk:local@cbfs";
property name="s3Disk"    inject="disk:s3@cbfs";
```

### Get a Disk Fluently

```js
var disk = diskService.get( "s3" )    // by config name
var disk = diskService.getDefault()   // configured default
```

## Production Patterns

### Write / Read / Delete

```js
// Write text
disk.put( "documents/readme.txt", "Hello World" )

// Write binary
disk.put( "images/photo.jpg", fileReadBinary( localPath ) )

// Read text
var content = disk.get( "documents/readme.txt" )

// Read binary
var bytes = disk.getAsBinary( "images/photo.jpg" )

// Delete
disk.delete( "documents/readme.txt" )

// Existence check
if ( disk.exists( "documents/readme.txt" ) ) { ... }
```

### Copy & Move

```js
disk.copy( "originals/file.pdf", "archive/file.pdf" )
disk.move( "temp/upload.pdf",    "docs/upload.pdf" )
```

### File Upload Handler

```js
function upload( event, rc, prc ) {
    var upload = fileUpload(
        getTempDirectory(),
        "file",
        "image/jpeg,image/png,application/pdf",
        "makeUnique"
    )

    var storedPath = "uploads/#upload.serverFile#"

    // Store to S3 in production, local disk in dev
    diskService.get( "s3" ).put(
        storedPath,
        fileReadBinary( upload.serverDirectory & "/" & upload.serverFile )
    )

    // Clean up temp file immediately
    fileDelete( upload.serverDirectory & "/" & upload.serverFile )

    prc.filePath = storedPath
}
```

### Generate URLs

```js
// Public URL
var url = disk.url( "images/photo.jpg" )

// Temporary signed URL (S3 pre-signed)
var signedUrl = disk.temporaryUrl( "documents/private.pdf", 60 ) // 60 minutes
```

### Directory Listing

```js
// List files at path
var files       = disk.files( "uploads/" )      // array of file paths
var directories = disk.directories( "uploads/" )

// Recursive listing
var allFiles = disk.allFiles( "uploads/" )
```

### File Metadata

```js
var info = disk.info( "uploads/myfile.pdf" )
// Returns: { size, lastModified, type, path, ... }

var size     = disk.size( "uploads/myfile.pdf" )
var modified = disk.lastModified( "uploads/myfile.pdf" )
```

## Best Practices

- **Use named disks** — always reference `"s3"` or `"local"` by config name, never instantiate providers directly
- **Abstract disk choice** — store disk name in config or env: `getSystemSetting( "STORAGE_DISK", "local" )`
- **Validate uploads** before storing — check file type, size, and sanitize the filename
- **Delete temp files** immediately after reading the binary content
- **Never expose raw local paths** to users — use the `url()` API
- **Use signed URLs** for private S3 objects rather than making buckets public

## Documentation

- cbfs: https://cbfs.ortusbooks.com
