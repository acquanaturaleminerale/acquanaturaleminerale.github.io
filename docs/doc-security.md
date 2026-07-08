# Configuration

The SAML integration uses a combination of environment variables and local deployment files to configure the Service Provider.

Configuration is loaded during application startup and remains static for the lifetime of the running application process.

Changes to SAML configuration require an application restart.

The configuration consists of:

- Service Provider identity and endpoints;  
- Identity Provider metadata;  
- Service Provider signing credentials;  
- SAML runtime dependencies;  
- Environment-specific runtime settings.  

---

## Service Provider information

The Service Provider configuration defines Exerplaza as a SAML participant.

The following values are configured:

### Entity identifier

The Service Provider entity identifier uniquely identifies Exerplaza to the Identity Provider.

The current implementation derives the entity identifier from the configured base URL:

```
{SAML_BASE_URL}/saml/sp
```

Example:

```
https://exerplaza.example.com/saml/sp
```

---

### Assertion Consumer Service endpoint

The Assertion Consumer Service (ACS) endpoint receives SAML responses returned by the Identity Provider.

The current endpoint is:

```
{SAML_BASE_URL}/saml/acs
```

The endpoint uses the HTTP POST SAML binding.

---

### Service Provider certificates

The Service Provider uses an X.509 certificate and private key for signing SAML authentication requests.

Configured files:

```
backend/saml_configuration/certs/sp-cert.pem  
backend/saml_configuration/certs/sp-key.pem  
```

The certificate and private key must belong to the same key pair.

The application validates the certificate material during SAML initialization before enabling authentication.

Certificate rotation is currently a manual process.

During certificate replacement, the new certificate and key pair should be added to the configuration alongside the existing pair. Both credential pairs may temporarily exist at the same time during the transition period.

After the Identity Provider has been updated and the new certificate is active, the old certificate and key can be removed from the configuration.

Certificate changes require an application restart before the new credentials are loaded.

Certificate rotation is expected to be an infrequent operational task and should only be performed when required by certificate expiration, security policy, or credential compromise.

---

## Identity Provider information

The Identity Provider configuration is provided through locally stored metadata.

Current metadata source:

```
backend/saml_configuration/metadata/idp_metadata.xml
```

The metadata file contains:

- Identity Provider entity information;  
- SAML endpoints;  
- trusted signing certificates.  

The current implementation does not dynamically retrieve metadata from the Identity Provider.

Changes to IdP metadata require replacing the metadata file and restarting the application.

The current implementation supports a single configured Identity Provider.

---

## Environment variables

The SAML integration reads runtime-specific values from environment variables.

### Required variables

```
SAML_BASE_URL=  
SAML_SECRET=  
```

`SAML_BASE_URL` defines the public URL used to generate Service Provider identifiers and endpoints.

`SAML_SECRET` provides the secret value used by PySAML2 for internal SAML state handling.

---

### SAML attribute mapping


The current implementation relies on PySAML2’s default attribute handling when extracting identity attributes from the SAML response.

Exerplaza currently requires an email attribute to be present in the validated SAML identity data. That email value is used to resolve the local Exerplaza user account.

Current behaviour:

- email is the primary identity attribute used by Exerplaza;  
- no local user provisioning is performed from SAML assertions;  
- if the SAML response does not provide a usable email value, authentication fails;  
- if no existing Exerplaza user matches the email address, authentication fails.  

If a production Identity Provider uses different attribute naming conventions, the SAML attribute mapping configuration may need to be adjusted so that Exerplaza can reliably obtain the user email address.

In the current implementation, a successful SAML login requires a usable email attribute in the validated SAML response because email is the only supported local user lookup key.

---

### Optional variables

```
SAML_DEBUG=false  
SAML_LOG_LEVEL=INFO  
```

`SAML_DEBUG` enables additional PySAML2 debugging output.

`SAML_LOG_LEVEL` controls SAML-related logging verbosity.

Available levels:

```
DEBUG
INFO
WARNING
ERROR
CRITICAL
```

Debug logging should not be enabled in production environments because SAML processing logs may contain sensitive diagnostic information.

---

## External dependencies

The SAML integration requires the following external dependency:

### xmlsec1

PySAML2 uses xmlsec1 for XML signature operations.

The application validates that the binary is available during startup.

Missing xmlsec1 prevents SAML initialization.

---

## validation

SAML configuration validation is performed in two stages.

### Startup validation

Performed during application startup.

Checks:

- required environment variables;  
- logging configuration;  
- xmlsec1 availability.  

---

### Runtime SAML validation

Performed before initializing the Service Provider.

Checks:

- Identity Provider metadata availability;  
- Service Provider certificate availability;  
- private key availability;  
- certificate and private key matching;  
- SAML secret availability.  

This separation allows basic deployment errors to fail early while keeping environment-dependent checks close to SAML initialization.

# Security Considerations

## Trust Model

The SAML integration relies on several trusted components:

- The configured Identity Provider metadata.
- The Identity Provider signing certificates contained in metadata.
- The Service Provider private key used for signing.
- The application database storing authentication transaction state.

Identity Provider metadata is loaded locally and is considered trusted configuration.

The implementation does not dynamically fetch metadata during runtime. Changes to Identity Provider metadata require replacing the local metadata file and restarting the application.

PySAML2 is responsible for protocol-level validation, including signature and assertion validation. Exerplaza remains responsible for application-level authentication decisions.

Protocol-level time validation of SAML assertions, including checks such as assertion validity windows and related clock-skew handling, is delegated to PySAML2 rather than implemented directly in the Exerplaza application layer.

The Service Provider private key must be protected as deployment-sensitive secret material. Unauthorized access to this key could allow an attacker to create SAML messages appearing to originate from the Service Provider.

## Authentication transaction tracking

SAML authentication transactions are tracked using a persistent database-backed request store.

Each SAML login attempt creates a temporary authentication transaction containing:

- the generated SAML AuthnRequest identifier;
- the RelayState value associated with the login attempt;
- the request expiration timestamp;
- the consumption status of the transaction.

The transaction store exists to correlate SAML responses returned by the Identity Provider with authentication requests originally created by Exerplaza.

The implementation does not rely on PySAML2's default in-memory request state handling because the application runs with multiple workers. Process-local state would not guarantee that the same worker handles both the authentication request and the returned SAML response.

Persistent storage ensures that authentication transaction validation works consistently across application workers.

Authentication transaction records represent temporary security state only. They are not user sessions and do not contain application authorization state.

---

## Replay protection

The implementation prevents replay of previously accepted SAML authentication responses through single-use authentication transactions.

Before creating an authenticated application session, the returned SAML response must reference an existing authentication transaction that:

- was created by Exerplaza;
- has not expired;
- has not already been consumed.

Consumption of the authentication transaction is performed atomically at the database level.

This prevents concurrent authentication attempts from successfully processing the same SAML response multiple times.

Once a transaction has been consumed, subsequent attempts to use the same request identifier are rejected.

Expired transactions are invalid and cannot be used to create authenticated sessions.

---

## RelayState validation

RelayState is used to preserve the intended application destination across the external Identity Provider authentication flow.

Before starting authentication, the requested redirect destination is validated by Exerplaza.

Only approved internal application destinations are accepted.

The stored RelayState value is recovered from the validated authentication transaction after successful SAML response processing.

This prevents attackers from using the SAML login flow as an open redirect mechanism.

RelayState handling is performed by the application layer rather than delegated entirely to PySAML2.

---

## Certificate usage

The Service Provider uses an X.509 certificate and private key for SAML signing operations.

The private key is used to sign SAML authentication requests sent to the Identity Provider.

The public certificate is provided to the Identity Provider so that signed requests can be verified.

Certificate material is loaded from the local deployment environment and validated before SAML initialization.

The application verifies that:

- the certificate exists;
- the private key exists;
- the certificate is readable;
- the certificate matches the private key.

The private key must be protected as deployment-sensitive secret material. Unauthorized access could allow an attacker to impersonate the Service Provider.

Certificate rotation is currently a manual operational process.

---

## Application authentication boundary

The SAML integration authenticates users through the configured Identity Provider but does not replace the existing Exerplaza authentication system.

SAML is responsible for establishing the external identity of the user.

After successful SAML validation, Exerplaza remains responsible for:

- resolving the local user account;  
- checking application-specific access restrictions;  
- creating the application session;  
- enforcing application authorization rules.  

A valid SAML response alone does not create an application session. The user must also successfully pass Exerplaza-specific authentication checks.

This separation keeps external identity verification separate from application authorization and session management.

In the current implementation, successful SAML authentication is not sufficient on its own to grant access. Exerplaza must also be able to resolve a pre-existing local user account by the email address extracted from the SAML response. Missing email attributes, unknown users, or disabled users all result in authentication failure.

---

# Operational Logging

The SAML integration emits application-side diagnostic and audit-style log events through the Flask application logger.

Current logging covers key points in the authentication flow, including:

- metadata generation failures;
- login initialization failures;
- SAML ACS processing failures;
- successful SAML login initiation;
- successful SAML authentication completion;
- invalid, expired, or unknown authentication request handling;
- user resolution failures such as missing users or disabled users.

These logs are intended to support debugging, operational monitoring, and investigation of failed authentication attempts. Logged fields may include request identifiers, email addresses, internal user identifiers, and failure reason codes depending on the event.

The current implementation uses application logging for observability only. It does not currently define a dedicated audit logging subsystem, structured retention policy, or external security event pipeline as part of the SAML module itself.

# Testing

The SAML integration includes automated tests covering authentication flow correctness, security-sensitive behaviour, and failure handling.

Testing focuses on validating the application logic around SAML authentication rather than replacing external Identity Provider validation.

---

## Automated tests

The automated test suite covers:

- invalid SAML responses;
- expired authentication requests;
- replayed authentication responses;
- missing authentication transactions;
- invalid redirect destinations;
- disabled users;
- failed user resolution;
- authentication failures.

The tests validate that invalid authentication attempts fail safely without creating an authenticated application session.

---

### Component tests

Component tests validate individual SAML integration components.

Covered areas include:

- SAML configuration loading;
- route behaviour;
- authentication helpers;
- SAML transaction storage operations.

---

### Behavioural tests

Behavioural tests validate interactions between the different SAML integration components.

Covered scenarios include:

- creation of authentication requests;
- persistence of AuthnRequest tracking records;
- retrieval and validation of authentication transactions;
- single-use request consumption;
- response processing behaviour;
- application session creation after successful authentication.

---

## End-to-end testing

The complete SAML login flow has been validated using a local Keycloak Identity Provider environment.

The Keycloak environment is used exclusively for development and integration testing. It is maintained separately from the production deployment path and is not intended to be deployed as part of the Exerplaza production environment.

The end-to-end test validates the complete authentication lifecycle:

1. User starts authentication from the Exerplaza login page.  
2. Exerplaza generates a SAML AuthnRequest.  
3. The browser is redirected to the local Identity Provider.  
4. The user authenticates through the Identity Provider.  
5. The Identity Provider returns a SAMLResponse.  
6. Exerplaza validates the SAML response.  
7. The authentication transaction is consumed.  
8. The local user is resolved.  
9. The application session is created.  

The test environment validates the integration between Exerplaza, PySAML2, and an external SAML Identity Provider implementation.

---

# Certificate rotation

Certificate rotation is currently a manual operational process. The exact rotation procedure depends on how Service Provider certificate material is deployed and how the Identity Provider consumes updated Service Provider metadata or certificates.

The current implementation does not provide automated certificate rollover, automated renewal, or built-in certificate rotation orchestration. Rotation procedures should be validated against the target Identity Provider and deployment environment before production use.

Any certificate change requires updating the deployed Service Provider certificate material and restarting the application so the new configuration is loaded.

---

# Limitations

The current implementation provides a complete SAML authentication integration for the supported Exerplaza deployment scenario.

Known limitations:

- Single Identity Provider support only.
- Manual certificate rotation.
- Local metadata management.
- No SAML federation discovery.
- No automated certificate renewal.
- Additional production security review required before large-scale deployment.

---

# Future Improvements

Possible future extensions include:

- support for multiple Identity Providers;
- federation discovery support;
- additional security testing;
- production deployment validation;
- improved monitoring and logging.

---

# Maintenance Notes

When modifying the SAML implementation, particular attention should be given to:

- authentication request tracking logic;
- certificate management;
- Identity Provider configuration;
- authentication state handling.

Changes to these components should always be validated through automated tests and end-to-end authentication testing.

# Logging and operational diagnostics

The SAML integration emits application-side diagnostic logs for key Service Provider events and failures.

Examples include:

- SAML login initiation  
- ACS processing failures  
- missing or invalid authentication transactions  
- user resolution failures    
- disabled-user login attempts  
- successful SAML logins  
- metadata generation failures  

These logs are intended for developer and operational diagnostics rather than user-facing error handling. Log verbosity is influenced by the application logging configuration and, for PySAML2-specific logging, by the SAML debug and log-level settings.

Care should be taken not to expose sensitive SAML data in production logs. Debug logging should be enabled only when required for troubleshooting.

That’s enough. You don’t need a whole observability chapter.
