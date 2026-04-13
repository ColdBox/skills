---
name: cbjavaloader
description: >
  Use this skill when dynamically loading Java JAR libraries at runtime in a ColdBox application using
  cbjavaloader. Covers installation, classpath configuration, creating Java objects from JARs, reloading
  in development, and common use cases (PDF processing, Apache POI, image manipulation).
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBJavaloader Skill

## When to Use This Skill

Load this skill when:
- Loading third-party JAR files without restarting the CFML server
- Instantiating Java classes from dynamically loaded JARs
- Managing multiple JAR dependency paths
- Using Java libraries for PDF, Excel, image, or other processing
- Configuring reload behavior for development vs. production

## Installation & Configuration

```bash
box install cbjavaloader
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbjavaloader = {
        // Paths (relative to webroot) containing JAR files
        loadPaths = [ "lib", "jars", "libs/external" ],

        // Include server classpath (usually false — use explicit paths)
        loadColdFusionClassPath = false,

        // Reload classloader on each request — dev only, NEVER in production
        reloadOnEveryRequest = getSetting( "environment" ) == "development"
    }
}
```

## Core API

### Injection

```js
property name="javaloader" inject="loader@cbjavaloader";
```

### Create Java Objects

```js
// Create instance (calls default constructor)
var obj = javaloader.create( "com.example.MyClass" )

// Create and initialize with arguments
var reader = javaloader.create( "com.lowagie.text.pdf.PdfReader" )
    .init( expandPath( "/documents/file.pdf" ) )

// Get a static value / utility class
var constants = javaloader.create( "org.apache.poi.ss.usermodel.CellType" )
```

## Common Use Cases

### PDF Processing (iText / OpenPDF)

```js
var pdfReader = javaloader.create( "com.lowagie.text.pdf.PdfReader" )
    .init( expandPath( "/documents/report.pdf" ) )
var pages = pdfReader.getNumberOfPages()
```

### Excel Processing (Apache POI)

```js
var workbook = javaloader.create( "org.apache.poi.xssf.usermodel.XSSFWorkbook" )
    .init( fileReadBinary( expandPath( "/data/data.xlsx" ) ) )
var sheet = workbook.getSheetAt( 0 )
```

### Image Manipulation

```js
var bufferedImage = javaloader.create( "java.awt.image.BufferedImage" )
var imageIO       = javaloader.create( "javax.imageio.ImageIO" )
var img = imageIO.read( createObject( "java", "java.io.File" ).init( imagePath ) )
```

### HTTP Client (Apache HttpClient)

```js
var client   = javaloader.create( "org.apache.http.impl.client.CloseableHttpClient" )
var request  = javaloader.create( "org.apache.http.client.methods.HttpGet" ).init( url )
var response = client.execute( request )
```

## Best Practices

- **Never set `reloadOnEveryRequest = true` in production** — it destroys a new classloader each request
- **Use BoxLang's `this.javaSettings`** for Java libraries where possible — it's native and faster
- **Keep JARs in a dedicated folder** (`/lib/` or `/jars/`) tracked in source control or installed via a build step
- **Singleton the javaloader** — it is already a singleton by default via WireBox; don't re-create it
- **Prefer `cbfs` or native server APIs** for file-system operations rather than Java `java.io.File` directly

## Documentation

- cbjavaloader: https://github.com/coldbox-modules/cbjavaloader
