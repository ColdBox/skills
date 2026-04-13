---
name: cbmailservices
description: >
  Use this skill when sending email from a ColdBox/BoxLang application using cbmailservices. Covers
  installation, SMTP/Postmark/SendGrid protocol configuration, fluent mail builder API, body templates,
  token replacement, attachments, CC/BCC/replyTo, async sending, and production best practices.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBMailServices Skill

## When to Use This Skill

Load this skill when:
- Sending transactional or notification emails from handlers or services
- Configuring SMTP, Postmark, SendGrid, or Mailgun protocols
- Using HTML view templates for email bodies
- Adding attachments, inline images, CC, BCC, or replyTo
- Replacing body tokens `@firstName@` style
- Queuing emails asynchronously for background delivery
- Testing email sending without hitting a real mail server

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install cbmailservices
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbmailservices = {
        // Default protocol key (must match a key in protocols{})
        defaultProtocol = "smtp",

        // Token replacement delimiter (default @)
        tokenMarker = "@",

        protocols = {
            // SMTP (development / production)
            smtp = {
                class      = "cbmailservices.models.protocols.SMTPProtocol",
                properties = {
                    server   = getSystemSetting( "SMTP_HOST", "127.0.0.1" ),
                    port     = getSystemSetting( "SMTP_PORT",  587 ),
                    username = getSystemSetting( "SMTP_USER",  "" ),
                    password = getSystemSetting( "SMTP_PASS",  "" ),
                    useTLS   = true
                }
            },
            // Postmark
            postmark = {
                class      = "cbmailservices.models.protocols.PostmarkProtocol",
                properties = {
                    apiKey = getSystemSetting( "POSTMARK_API_KEY", "" )
                }
            },
            // Null mailer — swallows all mail (unit tests)
            null = {
                class      = "cbmailservices.models.protocols.NullProtocol",
                properties = {}
            }
        }
    }
}
```

> Always load credentials from environment variables; never hardcode them.

## Core API

### Injection

```js
property name="mailService" inject="MailService@cbmailservices";
```

### Simple Text / HTML Email

```js
mailService.newMail(
    to      = "[email protected]",
    from    = "[email protected]",
    subject = "Welcome!",
    body    = "<h1>Thanks for signing up</h1>",
    type    = "html"        // html | text
).send()
```

### Fluent Builder

```js
mailService.newMail()
    .setTo( "[email protected]" )
    .setCc( "[email protected]" )
    .setBcc( "[email protected]" )
    .setFrom( "[email protected]" )
    .setReplyTo( "[email protected]" )
    .setSubject( "Your Invoice #@invoiceId@" )
    .setBodyTokens( { invoiceId : invoice.getId() } )
    .setBodyTemplate( "/emails/invoice" )   // maps to /views/emails/invoice.cfm
    .setType( "html" )
    .addAttachment( expandPath( "/invoices/#invoice.getId()#.pdf" ) )
    .send()
```

### Body Templates

Create a ColdBox view at `/views/emails/welcome.cfm`:

```cfml
<!DOCTYPE html>
<html>
<body>
    <h1>Welcome, @firstName@!</h1>
    <p>Your account has been created. <a href="@verifyLink@">Verify your email</a>.</p>
</body>
</html>
```

Then send with token replacement:

```js
mailService.newMail()
    .setTo( user.getEmail() )
    .setFrom( "[email protected]" )
    .setSubject( "Welcome to MyApp" )
    .setBodyTemplate( "/emails/welcome" )
    .setBodyTokens( {
        firstName  : user.getFirstName(),
        verifyLink : event.buildLink( "auth.verify" ) & "?token=#token#"
    } )
    .send()
```

### Attachments

```js
mail
    .addAttachment( expandPath( "/storage/reports/report.pdf" ) )           // from disk
    .addAttachment( data : pdfBinary, filename : "report.pdf", type : "application/pdf" )  // binary
```

### Switch Protocol at Runtime

```js
mailService.newMail()
    .setProtocol( "postmark" )   // override module default
    .setTo( "[email protected]" )
    // ...
    .send()
```

## Best Practices

- **Use `NullProtocol` in tests** — prevents real emails from test suites
- **Switch protocol per environment** via `.env` variable: `MAIL_PROTOCOL=smtp|postmark|null`
- **Never hardcode SMTP credentials** — always use `getSystemSetting()`
- **Log send results** — `send()` returns a result object; check `getResultMessages()` for errors
- **Use body templates** for HTML emails — avoids string concatenation and is easier to design/maintain
- **Validate recipient addresses** before sending to avoid silent delivery failures
- **Avoid bulk sending in request thread** — dispatch a background job for large recipient lists

## Documentation

- cbmailservices: https://github.com/coldbox-modules/cbmailservices
