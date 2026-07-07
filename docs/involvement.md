# My Involvement

## Short overview (surface level)

During my involvement in the Exerplaza project, I designed and implemented a SAML-based Single Sign-On authentication system integrated into the existing Flask application. My work focused on introducing external Identity Provider authentication while preserving the existing architecture and authentication flow. The final result is a functional SAML prototype validated through automated tests and end-to-end testing with a local Identity Provider.

---

## Detailed summary 

My contribution to the Exerplaza project focused on designing and implementing a SAML-based authentication system while integrating it into an existing web application without disrupting the current architecture and authentication mechanisms.

The project started from an existing Flask application that used a traditional application-managed authentication flow based on user credentials and browser sessions. The objective was to introduce Single Sign-On capabilities through SAML authentication while respecting the constraints of the existing project, avoiding large-scale architectural changes that were not feasible within the project scope.

I began by analyzing the existing codebase, development environment, and authentication flow, then evaluated different approaches for implementing SAML authentication, including external SAML middleware/software solutions and direct integration through Python libraries. After comparing the available alternatives, I chose a Python-based approach and designed and implemented a Service Provider using PySAML2. PySAML2, a Python library for implementing SAML Service Providers and Identity Providers.

During development, I identified limitations in the library's default state management when used in the Exerplaza environment, particularly regarding the persistence of authentication state across application workers. To address these issues, I extended the implementation with a database-backed SAML request tracking system responsible for maintaining authentication state, validating requests, and preventing replay attacks.

The final result is a functional SAML authentication prototype integrated alongside the existing authentication system, validated through automated tests and end-to-end testing with a local Identity Provider. The implementation is documented through a complete setup guide that allows the environment to be reproduced. While the prototype successfully demonstrates the complete SAML authentication flow, additional security review, deployment validation, and production hardening would be required before being used in a production environment.

---

## Detailed involvement

### Project onboarding and analysis

At the beginning of my involvement, I had to become familiar with an existing project and its development environment. Since I was joining an already active codebase, my first objective was to understand the structure of Exerplaza, its technologies, and the workflow used by the development team.

I started by configuring the local development environment, learning the Docker-based setup, and resolving the initial configuration issues required to run the project locally. This phase allowed me to understand the relationship between the different services and how the application was structured.

After successfully setting up the environment, I analyzed the project architecture, reviewed the existing documentation, studied the directory structure, and examined the technologies used by Exerplaza. This analysis was necessary to understand where and how a new authentication mechanism could be integrated without disrupting the existing system.

---

### SAML research and architectural decisions

The initial task was to learn how to introduce SAML authentication into Exerplaza. Before starting implementation, I researched the SAML protocol, its authentication flow, its security requirements, and the available implementation strategies.

During this phase, I evaluated multiple approaches:

* using dedicated SAML authentication software such as Shibboleth or Apache Mellon;
* implementing the Service Provider directly inside the application using a Python library.

The software-based approach offered advantages in terms of standardization, maintainability, and reduced custom security logic. However, after discussing the available options with the development team and evaluating the project requirements, I decided that a direct PySAML2 integration was the most suitable approach for the current scope of the project.

This choice allowed the SAML functionality to remain directly integrated into Exerplaza, avoiding additional infrastructure complexity while providing full control over the authentication flow.

---

### Implementation

The implementation phase started with the creation of a minimal prototype using a single Identity Provider. This allowed me to validate my understanding of the SAML flow before considering a more complex federated setup.

I integrated PySAML2 into the project environment, configured the Service Provider, created the required SAML routes and templates, and implemented the authentication flow inside the existing Flask application.

During development, I also:

* introduced SAML-specific configuration through environment variables;
* created scripts for certificate generation and validation;
* integrated SAML configuration files into the existing project structure;
* implemented dedicated error handling for authentication failures.

After completing the initial implementation, I configured a local Keycloak Identity Provider to perform end-to-end testing of the complete authentication flow.

---

### Security and robustness improvements

During end-to-end testing, I discovered that the initial implementation, despite passing unit tests, was not sufficiently robust for realistic usage scenarios.

The main issue was related to PySAML2's internal handling of authentication request state. The library relied on temporary internal state that was not persistent across multiple workers or application restarts, causing valid SAML responses to be rejected in certain situations.

After investigating the problem, I redesigned this part of the system by moving request tracking responsibility into Exerplaza's database layer. I implemented persistent storage for SAML request information and added replay protection mechanisms by tracking request identifiers and validating their lifetime.

This improvement allowed the authentication flow to become independent from temporary library state and made the implementation more suitable for a multi-worker deployment environment.

---

### Testing and documentation

Throughout development, I used automated testing and manual end-to-end validation to verify the implementation.

I created multiple test iterations:

* initial functional tests to validate individual components;
* improved behavioural tests focused on the overall authentication flow;
* end-to-end tests using a local Keycloak Identity Provider.

The testing process was also fundamental for discovering weaknesses in the initial design and improving the reliability of the implementation.

After completing the prototype, I performed code cleanup and refactoring, added comments throughout the implementation, and produced documentation describing:

* the setup procedure;
* the required configuration;
* the local Identity Provider environment;
* the limitations of the current implementation.

---

### Final result and lessons learned

The final result of my work is a working SAML Service Provider prototype integrated into Exerplaza, capable of completing the authentication flow with an external Identity Provider and protected against replay attacks through persistent request tracking.

Beyond the technical implementation, this project provided experience in integrating new functionality into an existing software system, evaluating architectural trade-offs, working with authentication and security protocols, and adapting to unexpected challenges during development.

The most significant lesson was understanding that implementing an authentication protocol involves much more than following a specification: real-world integration requires careful consideration of state management, security, maintainability, testing, and compatibility with the surrounding system.
