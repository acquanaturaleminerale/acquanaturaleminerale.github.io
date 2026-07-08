# Documentation

## Overview

This module introduces SAML-based Single Sign-On authentication capabilities into Exerplaza.

The implementation acts as a SAML Service Provider (SP) integrated directly into the existing Flask application using PySAML2. It allows users to authenticate through an external Identity Provider (IdP) while integrating with the existing user management and session mechanisms. The SAML login flow is exposed through the existing login page by adding an additional entry point that redirects users to the SAML authentication endpoint.

The current implementation provides:

- SAML authentication through a configured Identity Provider  
- Service Provider metadata generation  
- SAML authentication request generation  
- Assertion Consumer Service (ACS) processing  
- Integration with the existing user session management  
- Database-backed SAML authentication request lifecycle tracking  
- Replay protection mechanisms through single-use request consumption and expiration validation  
- Local certificate generation and SAML environment setup utilities for testing and deployment preparation

The implementation currently supports a single configured Identity Provider and represents a complete authentication flow implementation within the supported project scope.

The SAML integration uses static configuration for the lifetime of a running application process. Environment settings and configuration paths are resolved during application startup. Basic startup validation is also performed at that stage. SAML-specific validation and PySAML2 Service Provider initialization occur when the SAML Service Provider is initialized. Once initialized, the Service Provider configuration is cached and remains static until the application restarts.

## Current limitations

The current implementation demonstrates the complete authentication flow implementation within the supported project scopes as a functional basis for further production validation. Before production use, additional validation and hardening would be required, including:

- Additional security hardening and review  
- Extended handling of edge cases and failure scenarios  
- Validation against the target production Identity Provider environment  
- Additional deployment-specific testing and operational validation  

The current implementation is designed around a single configured Identity Provider. Support for multiple Identity Providers or federation scenarios is not included in the current scope.

The current implementation does not provision or update users from SAML assertions. After a successful SAML authentication, Exerplaza resolves the user exclusively by email address against the existing internal user database. Authentication succeeds only if a matching local user already exists and is allowed to log in.

Time-based SAML assertion validation, including clock-related validity checks handled by the underlying SAML library, is not customized by the Exerplaza application layer in the current implementation.

Expired and unused authentication requests can be removed through the provided cleanup operation. The database does not automatically remove expired records; cleanup must be triggered by an external maintenance process or scheduled task.

## SAML Authentication Architecture 

The SAML authentication implementation introduces a Service Provider (SP) integration inside the existing Exerplaza Flask application. The integration extends the existing authentication flow by adding an external Identity Provider (IdP) authentication option while preserving the existing user and session management mechanisms.

The architecture consists of the following components:

- The existing Exerplaza authentication UI, which provides the entry point for SAML login.  
- The SAML Service Provider integration, which handles SAML endpoints and authentication flow coordination.  
- PySAML2, which provides SAML protocol handling and Service Provider functionality.  
- A database-backed AuthnRequest tracking store, which maintains authentication transaction state and provides replay protection.  
- The existing Exerplaza user and session management system, which remains responsible for application authentication state.  
- An external Identity Provider, which performs user authentication and returns SAML assertions.  

## SAML Authentication Flow

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant LoginPage as "Exerplaza Login Page"
    participant SamlModule as "Exerplaza SAML SP Integration"
    participant RequestStore as "SAML AuthnRequest Tracking Store"
    participant SessionSystem as "Exerplaza User / Session System"
    participant IdP as "Identity Provider (IdP)"

    Note over User,IdP: Access login page

    User->>Browser: Open login page
    Browser->>LoginPage: Request login page
    LoginPage-->>Browser: Render login form with SSO option

    Note over User,IdP: Initiate SAML authentication

    User->>Browser: Click "Log in with SSO"
    Browser->>SamlModule: GET /saml/login?next=...

    SamlModule->>SamlModule: Sanitize RelayState and generate SAML AuthnRequest through PySAML2
    SamlModule->>RequestStore: Store request_id and RelayState
    RequestStore-->>SamlModule: Request tracked

    SamlModule-->>Browser: Redirect to Identity Provider
    Browser->>IdP: Send SAML AuthnRequest

    Note over User,IdP: Authenticate with Identity Provider

    User->>IdP: Authenticate
    IdP-->>Browser: Return SAMLResponse form (HTTP POST)
    Browser->>SamlModule: POST /saml/acs

    Note over User,IdP: Process SAML response

    SamlModule->>SamlModule: Validate SAMLResponse
    SamlModule->>RequestStore: Consume request_id and recover RelayState
    RequestStore-->>SamlModule: RelayState

    SamlModule->>SessionSystem: Resolve local user
    SessionSystem-->>SamlModule: User found

    SamlModule->>SessionSystem: Create authenticated session
    SessionSystem-->>SamlModule: Session established

    SamlModule-->>Browser: Redirect to RelayState
    Browser-->>User: Return authenticated user

    Note right of SamlModule: SAML validation failures, expired requests, replayed requests, missing users, or disabled users redirect to the SAML error page instead of creating a session.
```

---

```mermaid
flowchart LR

    User["User"]
    Browser["Browser"]

    IdP["External Identity Provider<br/>(IdP)"]

    subgraph Exerplaza["Exerplaza Flask Application"]

        Login["Existing Login Page<br/>Authentication UI"]

        subgraph SAML["SAML Service Provider Integration"]

            Routes["SAML Endpoints<br/>/saml/login<br/>/saml/acs<br/>/saml/metadata"]

            PySAML["PySAML2 SAML Library"]

            Config["SAML Configuration<br/>Certificates<br/>Metadata<br/>Environment Settings"]

            Tracking["SAML AuthnRequest<br/>Tracking Store"]
        end

        UserSession["Existing User and<br/>Session Management"]
    end


    User --> Browser
    Browser --> Login

    Login --> Routes

    Routes --> PySAML

    Config --> PySAML

    PySAML --> IdP

    Routes --> Tracking

    Routes --> UserSession
```

---
