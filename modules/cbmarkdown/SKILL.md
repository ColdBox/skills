---
name: cbmarkdown
description: >
  Use this skill when converting Markdown to HTML in a ColdBox/BoxLang application using the cbmarkdown
  module. Covers installation, injecting the Processor, converting Markdown strings to HTML, using in views,
  supported syntax (GFM, tables, code blocks), and security considerations for user-generated Markdown.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBMarkdown Skill

## When to Use This Skill

Load this skill when:
- Rendering blog post bodies, documentation, or comments written in Markdown
- Converting Markdown to HTML in a service or view
- Combining cbmarkdown with cbantisamy to safely render user-submitted Markdown

## Installation

```bash
box install cbmarkdown
```

## Core API

### Injection

```js
property name="markdown" inject="Processor@cbmarkdown";
```

### Convert to HTML

```js
// Simple conversion
var html = markdown.toHTML( "# Hello **World**" )

// Multiline content
var content = "## Section Title

Paragraph text with **bold** and *italic*.

- Item one
- Item two

```javascript
console.log('code block')
```
"
var html = markdown.toHTML( content )
```

## Production Patterns

### In a Service / Handler

```js
class PostService {
    @inject("Processor@cbmarkdown")
    property name="markdown";

    function renderPost( required post ) {
        return {
            id      : post.getId(),
            title   : post.getTitle(),
            body    : markdown.toHTML( post.getBody() ),
            excerpt : markdown.toHTML( post.getExcerpt() )
        }
    }
}
```

### In a View

```cfml
<!--- Blog post body --->
<article>
    <h1>#encodeForHTML( prc.post.getTitle() )#</h1>
    <div class="post-body">
        #markdown.toHTML( prc.post.getBody() )#
    </div>
</article>
```

### Safe Rendering of User-Submitted Markdown

Markdown to HTML can still produce XSS vectors. Sanitize after converting:

```js
class CommentService {
    @inject("Processor@cbmarkdown")
    property name="markdown";
    @inject("AntiSamy@cbantisamy")
    property name="antiSamy";

    function renderComment( required string raw ) {
        // Convert Markdown → HTML, then sanitize the HTML output
        var html = markdown.toHTML( raw )
        return antiSamy.withPolicy( "relaxed" ).clean( html )
    }
}
```

## Supported Markdown Syntax

| Feature | Syntax |
|---------|--------|
| Headers | `# H1`, `## H2`, `### H3` |
| Bold | `**bold**` |
| Italic | `*italic*` |
| Unordered list | `- item` |
| Ordered list | `1. item` |
| Link | `[text](url)` |
| Image | `![alt](url)` |
| Fenced code | ` ```lang ``` ` |
| Inline code | `` `code` `` |
| Table (GFM) | `\| col \| col \|` |
| Blockquote | `> quote` |
| Horizontal rule | `---` |

## Best Practices

- **Always sanitize user-submitted Markdown** after conversion using cbantisamy
- **Do not sanitize trusted author content** (CMS editors, admins) — only user-generated input
- **Cache rendered HTML** for expensive/large documents rather than re-rendering on each request
- **Use `encodeForHTML()`** for plain title/metadata fields; reserve `toHTML()` for body fields only

## Documentation

- cbmarkdown: https://github.com/coldbox-modules/cbmarkdown
- Flexmark (underlying parser): https://github.com/vsch/flexmark-java
