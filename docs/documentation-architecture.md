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
