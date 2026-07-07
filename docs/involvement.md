# My Involvement

## Short overview (surface level)

During my involvement in the Exerplaza project, I designed and implemented a SAML-based Single Sign-On authentication system integrated into the existing Flask application. My work focused on introducing external Identity Provider authentication while preserving the existing architecture and authentication flow. The final result of my work is a functional SAML Service Provider prototype integrated into Exerplaza, capable of completing the authentication flow with a single external Identity Provider and reducing the risk of reused authentication requests through persistent SAML request tracking.

---

## Detailed summary 

My contribution to the Exerplaza project focused on designing and implementing a SAML-based authentication system while integrating it into an existing web application without disrupting the current architecture and authentication mechanisms.

The project started from an existing Flask application that used a traditional application-managed authentication flow based on user credentials and browser sessions. The objective was to introduce Single Sign-On capabilities through SAML authentication while respecting the constraints of the existing project, avoiding large-scale architectural changes that were not feasible within the project scope.

I began by analyzing the existing codebase, development environment, and authentication flow, then evaluated different approaches for implementing SAML authentication, including external SAML middleware/software solutions and direct integration through Python libraries. After comparing the available alternatives, I chose a Python-based approach and designed and implemented a Service Provider using PySAML2, a Python library providing SAML Service Provider functionality.

During development, I identified limitations in the library's default state management when used in the Exerplaza environment, particularly regarding the persistence of authentication state across application workers. To address these issues, I extended the implementation with a database-backed SAML request tracking system responsible for maintaining authentication state, validating requests, and preventing replay attacks.

The final result is a functional SAML authentication prototype integrated alongside the existing authentication system, validated through automated tests and end-to-end testing with a local Identity Provider. The implementation is documented through a complete setup guide that allows the environment to be reproduced. While the prototype successfully demonstrates the complete SAML authentication flow, additional security review, deployment validation, and production hardening would be required before being used in a production environment.

---

## Detailed involvement

!!! note:

    The following detailed involvement log is provided as supplementary documentation of the development process.
    The previous sections summarize the main contributions, decisions, and final results.

### Project onboarding and analysis

At the beginning of my involvement, I had to become familiar with an existing project and its development environment. Since I was joining an already active codebase, my first objective was to understand the structure of Exerplaza, the technologies used by the project, and the workflow followed by the development team.

The first phase focused on setting up and understanding the local development environment. I configured the Docker-based setup, resolved initial environment and configuration issues, and learned how the different services composing the application interacted with each other. This process allowed me to gain familiarity with the project's development workflow and the infrastructure required to run Exerplaza locally.

Once the environment was correctly configured, I analyzed the existing codebase by reviewing the available documentation, studying the project structure, examining the application architecture, and understanding the existing authentication flow. This analysis was necessary to identify where and how a new authentication mechanism could be introduced while preserving the existing system and avoiding unnecessary architectural changes.

---

### SAML research and architectural decisions

The initial task was to determine how SAML authentication could be introduced into Exerplaza. Before beginning implementation, I studied the SAML protocol, its authentication flow, the role of Service Providers and Identity Providers, and the security requirements involved in a correct implementation.

I first evaluated possible implementation strategies and available tools. Based on the initial analysis and discussions with the development team, I identified two main approaches: introducing an external SAML authentication layer through dedicated software, or implementing the Service Provider directly inside the existing application using a Python library.

Several Python libraries were considered, including Python3-SAML and django-saml2, but they were discarded due to framework incompatibility, project mismatch, or limitations regarding the required Service Provider implementation. PySAML2, a Python library providing SAML Service Provider functionality, was selected as the foundation of the implementation.

In parallel, I evaluated an alternative architecture based on dedicated SAML authentication software, such as Shibboleth or Apache Mellon, acting as an external authentication layer. This approach provided several advantages, including the use of mature enterprise-oriented software, standardized federation support, reduced responsibility for implementing SAML-specific security logic, and easier long-term maintenance.

The different approaches were evaluated through discussions with the development team and external technical guidance. The external authentication layer approach offered advantages in terms of federation support, reduced implementation responsibility, and long-term maintainability. However, integrating it into Exerplaza would have required significant changes to the existing authentication flow, session management, and deployment architecture.

Considering the existing Flask-based architecture, available development scope, and the need to integrate SAML alongside the current authentication system, I selected a direct Python-based implementation using PySAML2. This approach allowed SAML authentication to be introduced with fewer architectural changes while maintaining compatibility with the existing application.

---

### Implementation

The implementation phase started with the creation of a minimal prototype using a single Identity Provider. This approach allowed me to validate my understanding of the SAML authentication flow before considering additional complexity related to multiple Identity Providers and federation scenarios.

I integrated PySAML2 into the project environment, configured the Service Provider, created the required SAML routes and templates, and implemented the authentication flow inside the existing Flask application. The implementation was introduced as an additional authentication option, leaving the existing authentication mechanisms unchanged. During this phase, I adapted the library configuration to match the existing project structure and deployment environment while maintaining compatibility with the existing authentication system.

During development, I also:

introduced SAML-specific configuration through environment variables;  
created scripts for certificate generation and validation;  
integrated SAML configuration files into the existing project structure;  
implemented dedicated error handling for authentication failures.  

After completing the initial implementation, I learned how to configure and use a local Keycloak Identity Provider for end-to-end testing of the complete authentication flow. While the initial implementation successfully passed automated tests, end-to-end testing revealed a limitation related to the way PySAML2 handled internal authentication state.

PySAML2's default request state handling relied on internal state storage that was not suitable for the Exerplaza deployment environment, particularly due to the use of multiple uWSGI workers. As a result, valid SAML responses could be rejected because the authentication state created during the initial request was not consistently available during the response validation phase.

To address this limitation, I reorganized the implementation into separate modules responsible for SAML services, request tracking, configuration handling, and route management. I then implemented a database-backed SAML request tracking system, moving authentication transaction management from transient library state to persistent storage. This ensured consistency across different workers while providing additional protections through request identifier tracking, expiration validation, and single-use consumption of authentication requests.

After integrating this solution, I corrected the remaining implementation issues, completed end-to-end testing with the local Identity Provider, and created a second iteration of automated tests focused on validating overall system behaviour and interactions between components rather than only testing individual functions. Finally, I performed a code refactoring phase and added documentation comments throughout the implementation.

---

### Security and robustness improvements

The implementation includes several mechanisms to improve the reliability and security of the SAML authentication flow.

Persistent SAML request tracking prevents authentication requests from depending on temporary library-managed state and allows validation across multiple application workers. Requests are validated through unique identifiers, expiration timestamps, and single-use consumption, reducing the risk of replaying previously completed authentication transactions.

Additional validation is provided through automated tests covering authentication failures, invalid sessions, disabled users, unsafe redirects, and other edge cases.

The current implementation represents a functional prototype supporting authentication through a single Identity Provider. Before being considered production-ready, further validation would be required with the target Identity Provider environment, the production deployment configuration, and a broader security review.

---

### Testing and documentation

Throughout development, I used automated testing and manual end-to-end validation to verify the correctness and reliability of the implementation.

The testing process evolved through multiple iterations:

initial tests focused on validating individual routes and components;  
improved behavioural tests focused on interactions between different parts of the authentication flow;  
end-to-end validation using a local Keycloak Identity Provider.  

These iterations progressively increased confidence in the implementation by moving from isolated component validation to verification of the complete authentication flow.

After completing the prototype, I performed a final refactoring and cleanup phase, added documentation comments throughout the implementation, and documented:

the setup procedure;  
the required SAML configuration;  
the local Identity Provider environment;  
the current scope and limitations of the implementation.  

---

### Final result and lessons learned

The final result of my work is a functional SAML Service Provider prototype integrated into Exerplaza, capable of completing the authentication flow with a single external Identity Provider and implementing replay protection for authentication requests through persistent SAML request tracking.

Beyond the technical implementation, this project provided experience in integrating new functionality into an existing software system, evaluating architectural trade-offs, working with authentication and security protocols, and adapting to unexpected challenges during development.

The most significant lesson was understanding that implementing an authentication protocol involves much more than following a specification: real-world integration requires careful consideration of state management, security, maintainability, testing, and compatibility with the surrounding system.
