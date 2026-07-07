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

At the beginning of my involvement, I had to become familiar with an existing project and its development environment. Since I was joining an already active codebase, my first objective was to understand the structure of Exerplaza, the technologies used by the project, and the workflow followed by the development team.

The first phase focused on setting up and understanding the local development environment. I configured the Docker-based setup, resolved initial environment and configuration issues, and learned how the different services composing the application interacted with each other. This process allowed me to gain familiarity with the project's development workflow and the infrastructure required to run Exerplaza locally.

Once the environment was correctly configured, I analyzed the existing codebase by reviewing the available documentation, studying the project structure, examining the application architecture, and understanding the existing authentication flow. This analysis was necessary to identify where and how a new authentication mechanism could be introduced while preserving the existing system and avoiding unnecessary architectural changes.

---

### SAML research and architectural decisions

The initial task was to determine how SAML authentication could be introduced into Exerplaza. Before beginning implementation, I studied the SAML protocol, its authentication flow, the role of Service Providers and Identity Providers, and the security requirements involved in a correct implementation.

I first evaluated possible implementation strategies and available tools. Based on the initial analysis and discussions with the development team, I identified two main approaches: introducing an external SAML authentication layer through dedicated software, or implementing the Service Provider directly inside the existing application using a Python library.

Several Python libraries were considered, including Python3-SAML and django-saml2, but they were discarded due to framework incompatibility, project mismatch, or limitations regarding the required Service Provider implementation. PySAML2 was selected as the most suitable library because it provided a SAML Service Provider implementation compatible with the existing Flask and uWSGI-based application environment.

In parallel, I evaluated an alternative architecture based on dedicated SAML authentication software, such as Shibboleth or Apache Mellon, acting as an external authentication layer. This approach provided several advantages, including the use of mature enterprise-oriented software, standardized federation support, reduced responsibility for implementing SAML-specific security logic, and easier long-term maintenance.

The different approaches were evaluated through discussions with the development team and with external technical guidance. These discussions helped identify the strengths and limitations of each solution, particularly regarding maintainability, integration complexity, and compatibility with the existing project architecture.

However, after evaluating the alternatives against the project constraints, I concluded that integrating an external authentication layer would have required significant changes to the existing application architecture. Exerplaza already handled user sessions, authentication state, and user management internally, meaning that introducing an additional authentication component through middleware or external software would have required substantial integration logic and modifications to the existing request and session flow, particularly due to overlapping responsibilities with the existing Flask-based architecture.

Additionally, introducing an external authentication layer would have increased the complexity of the development environment by requiring contributors to understand and maintain another infrastructure component alongside the existing application stack.

After evaluating both approaches against the constraints of the existing project, including the application architecture, deployment environment, infrastructure constraints, and available development scope, I selected a direct Python-based implementation using PySAML2. This approach allowed SAML authentication to be introduced alongside the existing authentication system while avoiding large architectural changes that were not feasible within the project timeframe.

---

### Implementation

The implementation phase started with the creation of a minimal prototype using a single Identity Provider. This approach allowed me to validate my understanding of the SAML authentication flow before introducing additional complexity related to federation and multiple Identity Providers.

I integrated PySAML2 into the project environment, configured the Service Provider, created the required SAML routes and templates, and implemented the authentication flow inside the existing Flask application. During this phase, I adapted the library configuration to match the existing project structure and deployment environment while maintaining compatibility with the existing authentication system.

During development, I also:

introduced SAML-specific configuration through environment variables;
created scripts for certificate generation and validation;
integrated SAML configuration files into the existing project structure;
implemented dedicated error handling for authentication failures.

After completing the initial implementation, I learned how to configure and use a local Keycloak Identity Provider for end-to-end testing of the complete authentication flow. While the initial implementation successfully passed automated tests, end-to-end testing revealed a limitation related to the way PySAML2 handled internal authentication state.

PySAML2's default request state handling relied on internal state storage that was not suitable for the Exerplaza deployment environment, particularly due to the use of multiple uWSGI workers. As a result, valid SAML responses could be rejected because the authentication state created during the initial request was not consistently available during the response validation phase.

The implementation was progressively reorganized into separate modules responsible for SAML services, request tracking, configuration handling, and route management, improving separation of responsibilities and maintainability.

To address this limitation, I implemented a database-backed SAML request tracking system. This moved the management of SAML authentication transactions from transient application state to persistent storage, ensuring consistency across different workers. The system also provides additional protection mechanisms by tracking request identifiers and validating request expiration. The system also provides additional protection mechanisms by tracking request identifiers, validating request expiration, and preventing reuse of previously consumed authentication requests.

After integrating this solution, I corrected the remaining implementation issues, completed end-to-end testing with the local Identity Provider, and created a second iteration of automated tests focused on validating overall system behaviour and interactions between components rather than only testing individual functions. Finally, I performed a code refactoring phase, added documentation comments, and completed a functional SAML authentication prototype integrated alongside the existing Exerplaza authentication system.

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
