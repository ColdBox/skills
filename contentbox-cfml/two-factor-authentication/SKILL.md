---
name: contentbox-cfml-two-factor-authentication
description: "Use this skill when implementing or customizing ContentBox two-factor authentication providers, trusted device flows, enrollment/verification UX, enforcement policies, and provider lifecycle handling."
applyTo: "**/*.{cfc,cfm,cfml}"
---

# ContentBox Two-Factor Authentication (CFML)

Implement and extend two-factor authentication (2FA) in ContentBox CMS using CFML. ContentBox provides a pluggable 2FA system with provider interfaces, trusted device support, and global enforcement.

## 2FA Architecture

The 2FA system lives at `modules/contentbox/models/security/twofactor/`:

| Component | File | Purpose |
|-----------|------|---------|
| **Interface** | `ITwoFactorProvider.cfc` | Contract all 2FA providers must implement |
| **Base Class** | `BaseTwoFactorProvider.cfc` | Base class with injected services |
| **Service** | `TwoFactorService.cfc` | Provider registry and orchestration |

### TwoFactorService

The central service manages 2FA providers and settings:

```cfml
property name="twoFactorService" inject="twoFactorService@contentbox";

// Provider management
twoFactorService.registerProvider( provider )
twoFactorService.unRegisterProvider( name )
twoFactorService.getRegisteredProviders()
twoFactorService.getRegisteredProvidersWithDisplayNames()
twoFactorService.getProvider( name )

// Settings
twoFactorService.isForceTwoFactorAuth()
twoFactorService.getDefaultProvider()
twoFactorService.getDefaultProviderObject()
twoFactorService.getTrustedDeviceTimespan()
```

### 2FA Settings

Stored in the `cb_setting` table:

| Setting Key | Description |
|-------------|-------------|
| `cb_security_2factorAuth_force` | Force global 2FA enrollment (true/false) |
| `cb_security_2factorAuth_provider` | Default provider name |
| `cb_security_2factorAuth_trusted_days` | Trusted device cookie duration (days) |

### Trusted Device Cookie

```cfml
variables.TRUSTED_DEVICE_COOKIE = "contentbox_2factor_device"
```

When `allowTrustedDevice()` returns `true`, a cookie is set. If the user logs in from the same device within the trusted timespan, 2FA validation is skipped.

## ITwoFactorProvider Interface

All 2FA providers must implement this interface:

```cfml
interface {

	/**
	 * Get the internal name of a provider, used for registration, internal naming and more.
	 */
	function getName();

	/**
	 * Get the display name for the provider. Used in all UI screens.
	 */
	function getDisplayName();

	/**
	 * Returns HTML to display to the user for required two-factor fields.
	 */
	function getAuthorSetupForm( required author );

	/**
	 * Get the display help for the provider. Used in the UI setup screens for the author.
	 */
	function getAuthorSetupHelp( required author );

	/**
	 * Get the verification help for the provider. Used in the UI verification screen.
	 */
	function getVerificationHelp();

	/**
	 * If true, ContentBox will set a tracking cookie for the user's browser.
	 * If the user logs in and the device is within the trusted timespan,
	 * no two-factor authentication validation will occur.
	 */
	boolean function allowTrustedDevice();

	/**
	 * Send a challenge via the 2 factor auth implementation.
	 *
	 * @author The author to challenge
	 * @return struct:{ error:boolean, messages=string }
	 */
	struct function sendChallenge( required author );

	/**
	 * Verify a challenge for the specific user.
	 *
	 * @code   The verification code
	 * @author The author to verify challenge
	 * @return struct:{ error:boolean, messages:string }
	 */
	struct function verifyChallenge( required string code, required author );

	/**
	 * Called once a two factor challenge is accepted and valid.
	 * The user has completed validation and will be logged in.
	 *
	 * @code   The verification code
	 * @author The author to verify challenge
	 */
	function finalize( required string code, required author );

}
```

## Creating a Custom 2FA Provider

### Base Provider

Extend `BaseTwoFactorProvider` for auto-injected services:

```cfml
<!--- models/security/MyTOTPProvider.cfc --->
<cfcomponent
	extends="contentbox.models.security.twofactor.BaseTwoFactorProvider"
	singleton
	implements="contentbox.models.security.twofactor.ITwoFactorProvider"
>

	<cffunction name="init" access="public" returntype="MyTOTPProvider">
		<cfreturn this>
	</cffunction>

	<cffunction name="getName" access="public" returntype="string">
		<cfreturn "mytotp">
	</cffunction>

	<cffunction name="getDisplayName" access="public" returntype="string">
		<cfreturn "My TOTP Authenticator">
	</cffunction>

	<cffunction name="getAuthorSetupForm" access="public" returntype="string">
		<cfargument name="author" type="any">
		<cfreturn "
			<div class='form-group'>
				<label>Secret Key</label>
				<input type='text' name='totp_secret' class='form-control'
					value='#author.getTwoFactorSecret() ?: ''#'>
			</div>
		">
	</cffunction>

	<cffunction name="getAuthorSetupHelp" access="public" returntype="string">
		<cfargument name="author" type="any">
		<cfreturn "<p>Scan the QR code with your authenticator app.</p>">
	</cffunction>

	<cffunction name="getVerificationHelp" access="public" returntype="string">
		<cfreturn "<p>Enter the 6-digit code from your authenticator app.</p>">
	</cffunction>

	<cffunction name="allowTrustedDevice" access="public" returntype="boolean">
		<cfreturn true>
	</cffunction>

	<cffunction name="sendChallenge" access="public" returntype="struct">
		<cfargument name="author" type="any">
		<!--- Send the challenge (e.g., generate TOTP, send SMS, etc.) --->
		<cfreturn { error : false, messages : "Challenge sent successfully." }>
	</cffunction>

	<cffunction name="verifyChallenge" access="public" returntype="struct">
		<cfargument name="code" type="string">
		<cfargument name="author" type="any">

		<!--- Verify the code --->
		<cfset var isValid = verifyTOTP( arguments.code, arguments.author.getTwoFactorSecret() )>

		<cfif isValid>
			<cfreturn { error : false, messages : "Code verified." }>
		<cfelse>
			<cfreturn { error : true, messages : "Invalid code. Please try again." }>
		</cfif>
	</cffunction>

	<cffunction name="finalize" access="public" returntype="void">
		<cfargument name="code" type="string">
		<cfargument name="author" type="any">
		<!--- Called after successful validation --->
		<cfset log.info( "2FA finalized for author: #author.getUsername()#" )>
	</cffunction>

	<!--- Private helper --->
	<cffunction name="verifyTOTP" access="private" returntype="boolean">
		<cfargument name="code" type="string">
		<cfargument name="secret" type="string">
		<!--- TOTP verification logic --->
		<cfreturn true>
	</cffunction>

</cfcomponent>
```

## Registering a 2FA Provider

Register your provider in your module's `onLoad()`:

```cfml
<cffunction name="onLoad" access="public" returntype="void">
	<cfset var twoFactorService = wirebox.getInstance( "twoFactorService@contentbox" )>
	<cfset twoFactorService.registerProvider(
		wirebox.getInstance( "MyTOTPProvider@mymodule" )
	)>
</cffunction>
```

## 2FA Enforcement

### Global Enforcement

The `CheckForForceTwoFactorEnrollment` interceptor enforces 2FA globally:

- Runs on `preProcess` for admin requests
- Excludes security module events (login, logout, password reset)
- Redirects unenrolled users to `cbadmin/security/twofactorEnrollment/forceEnrollment`

### Provider Change Handling

The `UnenrollTwoFactorOnProviderChange` interceptor handles provider changes:

- When the default 2FA provider is changed in settings
- Unenrolls all users from 2FA (they must re-enroll with the new provider)

## Author 2FA Properties

The `Author` entity stores 2FA-related data:

```cfml
<!--- Author entity 2FA properties --->
<cfproperty name="twoFactorEnabled" ormtype="boolean" default="false">
<cfproperty name="twoFactorSecret" ormtype="string" length="500">
<cfproperty name="twoFactorProvider" ormtype="string" length="100">
```

## Checking 2FA Status

```cfml
property name="authorService" inject="authorService@contentbox";
property name="twoFactorService" inject="twoFactorService@contentbox";

// Check if global 2FA is forced
<cfif twoFactorService.isForceTwoFactorAuth()>
	<!--- All users must enroll --->
</cfif>

// Check if a specific author has 2FA enabled
<cfif author.getTwoFactorEnabled()>
	<!--- Author has 2FA enabled --->
</cfif>

// Get the default provider
<cfset var provider = twoFactorService.getDefaultProviderObject()>
```

## Built-in 2FA Providers

ContentBox ships with these providers:

| Provider | Description |
|----------|-------------|
| **TOTP** | Time-based One-Time Password (RFC 6238) — works with Google Authenticator, Authy, etc. |

## Best Practices

1. **Implement `ITwoFactorProvider`** — all methods are required
2. **Extend `BaseTwoFactorProvider`** — provides DI for `log`, `settingService`, `securityService`, `siteService`, `renderer`, `CBHelper`
3. **Return proper structs** — `sendChallenge()` and `verifyChallenge()` must return `{ error, messages }`
4. **Use `singleton` scope** — providers are instantiated once
5. **Register in `onLoad()`** — register providers after module configuration
6. **Support trusted devices** — implement `allowTrustedDevice()` for better UX
7. **Store secrets securely** — use encryption for sensitive data
8. **Provide clear help text** — `getAuthorSetupHelp()` and `getVerificationHelp()` guide users
9. **Handle errors gracefully** — return meaningful error messages
10. **Test enrollment flow** — verify the full enrollment and verification cycle

## Engine Compatibility

This skill targets **CFML engines** (Lucee 5+, Adobe ColdFusion 2018+). For BoxLang-specific syntax and features, see the BoxLang variant of this skill.
