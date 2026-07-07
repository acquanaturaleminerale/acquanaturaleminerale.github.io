# Setup

This guide explains how to initialize and configure the SAML Service Provider (SP) implementation used by Exerplaza.

The documented setup reproduces the development environment used for testing:
- Exerplaza as the Service Provider;
- PySAML2 as the SAML implementation library;
- Keycloak as the local Identity Provider.

Using another Identity Provider is supported, but it requires replacing the provided IdP metadata and adapting the Service Provider configuration accordingly.
The Keycloak setup described later is only a development convenience used during testing and is not a requirement of the SAML implementation.

!!! warning "Development environment only"

    This setup describes the current development and testing environment.
    The SAML implementation has not been validated for production deployment yet.

Do not use these settings directly in production. Before deploying to production, review:
- certificate management;
- secret handling;
- Identity Provider configuration;
- logging configuration;
- security settings.

## Prerequisites

### Use a branch that contains the SAML implementation

At the time of writing, the SAML implementation is available in the `simone-developement` branch.

Use the branch containing the latest SAML implementation before following this guide.

After switching to the correct branch, pull the latest changes before starting the setup process.

!!! note:

    Exerplaza’s current SAML implementation is based on PySAML2.
    If you need additional information about library behaviour or configuration, refer to the [PySAML2 documentation](https://pysaml2.readthedocs.io/).

---

## 1. Configure environment variables

Before starting the application, configure the SAML-related variables in the `.env` file.

Example:

```env
SAML_BASE_URL=http://127.0.0.1:54321
SAML_DEBUG=true
SAML_LOG_LEVEL=DEBUG
SAML_SECRET=0123456789
```

The initial `SAML_SECRET` value can be temporary. It will be replaced later with a cryptographically generated secret.

### Variable reference

| Variable         | Purpose                                                                                                                                  | Development value        |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| `SAML_BASE_URL`  | Public base URL of the Service Provider. It must match the URL used to access Exerplaza and the URL configured in the Identity Provider. | `http://127.0.0.1:54321` |
| `SAML_DEBUG`     | Enables PySAML2 debug mode. Useful during development, but generally not recommended in production.                                      | `true`                   |
| `SAML_LOG_LEVEL` | Controls the verbosity of SAML-related logs. Valid values are `DEBUG`, `INFO`, `WARNING`, `ERROR`, and `CRITICAL`.                       | `DEBUG`                  |
| `SAML_SECRET`    | Secret used by PySAML2 to sign and protect internal SAML state. Replace the temporary value after generating the final secret.           | generated later          |

Notes:

* `SAML_BASE_URL` must match the URL shown in the browser when accessing the Exerplaza home page.
* `SAML_DEBUG` can be set to either `true` or `false`.
* In production, `SAML_LOG_LEVEL=INFO` is recommended unless deeper debugging is required.

---

## 2. Start the Docker environment

Start the application containers:

```bash
docker compose up
```

The Docker environment should already contain the system dependencies and Python libraries required by the SAML implementation. If a required dependency is missing, the application will fail during startup validation or SAML initialization.

Starting the containers ensures that:

* the application environment is available;
* required services are running;
* SAML-related commands can be executed inside the `web` container.

!!! note

```
At this stage the SAML system is not fully configured yet. The endpoints may exist, but SAML authentication will not work until the remaining setup steps are completed.
```

---

## 3. Generate Service Provider certificates

Run the certificate generation script:

```bash
docker compose exec -it web bash exerplaza/backend/saml_configuration/scripts/cert_generator.sh
```

The script generates a new certificate and private key inside the `next/` subdirectory:

```text
exerplaza/backend/saml_configuration/certs/
└── next/
    ├── sp-cert.pem
    └── sp-key.pem
```

The certificate is used by the Service Provider to sign SAML requests.

If the configured Identity Provider requires signed requests, the SP certificate must also be registered inside the IdP configuration.

The script also performs validation checks to ensure that:

* the certificate can be read correctly;
* the certificate is not already expired;
* the private key matches the certificate.

At the end of the process, the script prints the location of the generated files and basic certificate details.

---

## 4. Move the generated certificate pair into the active certificate location

After generating the files, move or copy them from `certs/next/` into the active certificate directory expected by the application:

```text
exerplaza/backend/saml_configuration/certs/sp-cert.pem
exerplaza/backend/saml_configuration/certs/sp-key.pem
```

The current runtime configuration in `sp_settings.py` expects the certificate and private key at those exact paths.

This step is required to ensure that the generated certificate pair is actually used by the Service Provider.

---

## 5. Generate the SAML secret

Generate a cryptographically random secret:

```bash
docker compose exec -it web openssl rand -hex 32
```

Replace the value of `SAML_SECRET` in `.env` with the generated output.

Example:

```env
SAML_SECRET=<generated-value>
```

The secret should remain stable across deployments. Changing it invalidates existing SAML state and may break in-flight or persisted SAML-related data.

---

## 6. Configure Identity Provider metadata

### Default metadata file

The repository already contains an `idp_metadata.xml` file used during development with the local Keycloak Identity Provider. That file is only valid for that specific Keycloak configuration. The provided metadata file should work with the Keycloak configuration described later. However, it is recommended to replace it with the metadata provided by the actual Identity Provider, even when using the same IdP software.

If you are using a different Identity Provider, replace it with the correct metadata for your IdP.

The SAML response must contain an email attribute so make sure the idp provides one. If you are using the keycloak idp mentioned further below an email attribute is already configured in it.

Exerplaza uses this attribute to associate the authenticated SAML user with an existing local user. 

When using a different Identity Provider, ensure that:
- an email attribute is released;
- the attribute name matches the configured mapper;
- the value matches the user's Exerplaza account email.

If you do not have an idp yet you can follow the local keycloak idp setup guide further below or set up your own.

The Identity Provider metadata file must be provided by the Identity Provider administrator or generated from the IdP's metadata endpoint.

Place the Identity Provider metadata file inside:

```text
exerplaza/backend/saml_configuration/metadata/
```

The file must be named:

```text
idp_metadata.xml
```

The IdP metadata file is the only IdP-specific file required by the default setup.

When replacing the Identity Provider:
1. replace `idp_metadata.xml`;
2. ensure the IdP provides the attributes required by Exerplaza;
3. ensure `saml_sp_config.py` matches the IdP configuration.

---

## 7. Verify the Service Provider configuration

The Service Provider configuration is defined in:

```text
exerplaza/backend/saml_configuration/saml_sp_config.py
```

Review it and ensure that it matches the Identity Provider you intend to use.

The default configuration was tested against the local Keycloak Identity Provider used during development, so if you are using a different IdP you may need to adjust the configuration accordingly.

Important values to verify include:

* the ACS endpoint;
* the SP entity ID;
* signing requirements;
* certificate and key configuration;
* metadata location;
* any IdP-specific expectations required by your environment.

If you need to modify the SP configuration, refer to the [PySAML2 configuration documentation](https://pysaml2.readthedocs.io/en/latest/howto/config.html).

---

## 8. Restart the application

After updating the environment variables, certificates, metadata, or SAML configuration, restart the application so the new values are picked up correctly.

You can stop the running containers with:

```bash
docker compose down
```

and then start them again when needed.

This is necessary because the SAML configuration is loaded from environment variables and files, and the Service Provider configuration is cached by the Flask application once initialized.

---

# Verify successful setup

The setup is complete when:

- Exerplaza starts without SAML configuration errors;
- the SAML login option is visible;
- the user is redirected to the Identity Provider;
- authentication succeeds;
- the user is redirected back to Exerplaza;
- the authenticated user matches an existing Exerplaza account.

---

# Optional: Local Keycloak setup for development testing

This section is not required for production deployment.

It describes the Identity Provider environment used during development and automated testing.
You can use your own idp setup for testing.

## 1. Import the Keycloak Docker configuration

From the `simone-developement-keycloak` branch, import the `docker-compose.yml` file that includes the Keycloak service.

## 2. Rebuild and start the project

Run:

```bash
docker compose up --build
```

This rebuilds the project, adds the Keycloak service and its persistence volume, and installs the components required to run Keycloak locally.

Wait until Keycloak finishes booting before continuing.

## 3. Log in to Keycloak

Open:

`http://localhost:8080/`

Use the default credentials:

```text
username: admin
password: admin
```

You can change these credentials afterwards if needed.

## 4. Import the Keycloak realm configuration

The default Keycloak configuration used during development is available in the `simone-developement-keycloak` branch as:

```text
realm-export.json
```

The export contains:
- the SAML client representing Exerplaza;
- client scopes;
- attribute mappers;
- signing configuration;
- realm settings.

Import the realm export into Keycloak after logging in.

The export recreates the development realm configuration, including the SAML client, scopes, and mappings required for testing.

You can also configure your own Keycloak realm manually, but in that case make sure that:

* the realm is configured correctly for Exerplaza as a SAML Service Provider;
* the necessary client scopes and attribute mappings are present;
* the Service Provider configuration in Exerplaza matches the Keycloak configuration.

The SAML response must contain the user's email address.

The provided Keycloak configuration includes the required attribute mapping. If using another IdP, ensure that an email attribute is released and that it matches the attribute expected by Exerplaza. 

Refer to the keycloak documentation if configuring your own local keycloak idp [keycloak documentation](https://www.keycloak.org/documentation)

## 5. Export the Keycloak metadata

Once Keycloak is configured, open its SAML metadata endpoint and copy the generated metadata.

Keycloak exposes the SAML Identity Provider metadata through its descriptor endpoint:

```text
http://<keycloak-host>:<port>/realms/<realm-name>/protocol/saml/descriptor
```

if you are using the same keycloak setup used for testing it is found at:

```text
http://localhost:8080/realms/saml-test/protocol/saml/descriptor
```

Then replace the metadata file used by Exerplaza:

```text
exerplaza/backend/saml_configuration/metadata/idp_metadata.xml
```

Make sure the metadata includes the signing certificate expected by the Service Provider.

The default signing certifiacte in the keycloak idp is the one used during testing and should be replaced with your generated one during the setup process

## 6. Begin local IdP testing

At this point you can test the SAML flow locally against Keycloak.

A helper script for creating a local Exerplaza test user is available at:

```text
exerplaza/backend/saml_configuration/scripts/test_user.py
```

Run it inside the container:

```bash
docker compose exec -it web exerplaza/backend/saml_configuration/scripts/test_user.py
```

Make sure a Keycloak user with the same email address also exists, otherwise the SAML login will succeed at the IdP level but fail during Exerplaza’s local user lookup.

After completing the setup, access:

http://127.0.0.1:54321

and use the SAML login option.

The normal Exerplaza authentication system remains available. SAML is an additional authentication method.

---
# Changing SAML settings

To modify the Exerplaza Service Provider settings, edit:

```text
exerplaza/backend/saml_configuration/saml_sp_config.py
```

If you need to change runtime validation behaviour or file paths, also review:

```text
exerplaza/backend/saml_configuration/sp_settings.py
```

After changing the SAML configuration, restart the project so the new configuration is loaded correctly. Do not rely on hot changes while the application is already running.

