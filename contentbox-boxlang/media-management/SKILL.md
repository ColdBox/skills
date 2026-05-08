# ContentBox Media Management (BoxLang)

Work with the ContentBox media manager using BoxLang. Handle file uploads, image processing, media library organization, and media delivery.

## Media Manager Overview

The media manager is accessible via the admin at `cbadmin/mediamanager` and provides:

- File upload and management
- Image cropping and resizing
- Folder organization
- File type validation
- Media delivery via `__media` route

## Media Service

```boxlang
property name="mediaService" inject="mediaService@contentbox"

// Upload files
mediaService.upload( file, folder )

// Find media
mediaService.findBySlug( slug )
mediaService.findByPath( path )
mediaService.findAll( criteria )

// Delete media
mediaService.delete( media )

// Get media URL
mediaService.getMediaURL( media )
```

## Media Entity

```boxlang
// Media properties
media.getMediaID()
media.getSlug()
media.getFileName()
media.getFileExtension()
media.getFileSize()
media.getMimeType()
media.getFilePath()
media.getMediaURL()
media.getThumbnailURL()
media.getWidth()
media.getHeight()
media.getAltText()
media.getCreatedDate()
media.getModifiedDate()
```

## Media Delivery

Media is served via the `__media` route:

```
GET /__media/:slug
GET /__media/:slug/:size
```

### Image Sizes

ContentBox supports on-the-fly image resizing:

```boxlang
// In views
<img src="#{cb.mediaURL( media, 'thumbnail' )}#" alt="#{media.getAltText()}#">
<img src="#{cb.mediaURL( media, 'medium' )}#" alt="#{media.getAltText()}#">
<img src="#{cb.mediaURL( media, 'large' )}#" alt="#{media.getAltText()}#">
```

## File Browser Integration

ContentBox integrates with file browser modules for CKEditor and other editors:

```boxlang
// File browser module name (configurable)
property name="moduleSettings" inject="coldbox:moduleSettings:contentbox"
fileBrowserModule = moduleSettings.contentbox.filebrowser_module_name
```

## Media in Content

Media can be embedded in entry/page content:

```boxlang
// Image in content
<img src="#{cb.mediaURL( media )}#" alt="Description">

// With specific size
<img src="#{cb.mediaURL( media, 'thumbnail' )}#" alt="Description">
```

## CBHelper Media Methods

```boxlang
property name="cb" inject="CBHelper@contentbox"

// Get media URL
cb.mediaURL( media )
cb.mediaURL( media, "thumbnail" )
cb.mediaURL( media, "medium" )
cb.mediaURL( media, "large" )

// Get media path
cb.mediaPath( media )
```

## Media Storage

Media is stored using cbfs (ContentBox File System):

```boxlang
property name="diskService" inject="DiskService@cbfs"

// Get the contentbox disk
disk = diskService.get( "contentbox" )

// Disk configuration
// Path: expandPath( settingService.getSetting( "cb_media_directoryRoot" ) )
// URL: siteRoot & "/modules_app/contentbox-custom/_content"
```

## Best Practices

1. **Use `cb.mediaURL()`** for all media URL generation
2. **Set alt text** for accessibility
3. **Use appropriate sizes** — thumbnail, medium, large
4. **Validate file types** — check MIME types before processing
5. **Organize with folders** — use folder structure for large media libraries
6. **Handle missing media** — gracefully handle deleted or missing files
7. **Use cbfs for storage** — leverage the disk service for file operations
8. **Clean up orphaned media** — remove unused files periodically

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
