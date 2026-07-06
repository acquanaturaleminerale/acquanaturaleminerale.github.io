# Setup

## Initial Setup

## 0) Make sure you pull the correct branch

currently the main branch so the saml implementation is currently only under the simone-developement branch
make sure to git pull the branch.

## 1) Configure environment variables

Before starting the application, configure the SAML-related variables inside .env.
example:

```env
SAML_BASE_URL=http://127.0.0.1:54321
SAML_DEBUG=true
SAML_LOG_LEVEL=DEBUG
SAML_SECRET=0123456789 #temporary value
```

# Variable description
| Variable | Purpose | Development value |
|---|---|---|
| `SAML_BASE_URL` | Public URL of the Service Provider. Must match the URL exposed by Flask and the URL configured in the IdP. | `http://127.0.0.1:54321` |
| `SAML_DEBUG` | Enables PySAML2 internal debugging logs. Useful during development, should generally be disabled in production. | `true` |
| `SAML_LOG_LEVEL` | Controls the verbosity of SAML-related logs. | `DEBUG` |
| `SAML_SECRET` | Secret used by PySAML2. Generate a random value during initialization and replace it after setup. | generated value |

SAML_BASE_URL must match the one shown on browser when acessing the exerplaza home page

SAML_DEBUG can be either true or false

SAML_LOG_LEVEL can be  "DEBUG", "INFO", "WARNING", "ERROR" or "CRITICAL" for production INFO is raccomanded

At this stage SAML_SECRET can contain a temporary value. It will be replaced later after generating the final secret.

## 2) Start the docker environment

By default the system dependencies and the necessary libraries for the correct initalization of the saml implementation should be
already present within the container however the system will warn you if it they are not present.

Initialize the application containers:
```bash
docker compose up
```
This ensures that:

dependencies are installed;
the application environment exists;
required services are available.

Be aware that the saml system at this stage is not yet configured and will not run if you try to reach one if its endpoints.

## 3) Generate Service Provider certificates

execute:
```bash
docker compose exec -it web bash exerplaza/backend/saml_configuration/scripts/cert_generator.sh
```
the script generates inside certs in saml/configuration: 
```structure
saml_configuration/
└── certs/
    └── next/
        ├── sp-cert.pem
        └── sp-key.pem
```
The generated files are used by the Service Provider to sign SAML messages.

The script also performs validation checks to ensure:

the certificate is readable;
the certificate has not expired;
the private key matches the certificate.

The script will also tell you the location on generated files.

## 4) Extract generated files 

Extract generated files from next to certs.

This ensures the certificate/key pair is hooked up to the system.

## 5) Generate the SAML secret

Generate a cryptographically random secret:

docker compose exec -it web openssl rand -hex 32

Replace the value of SAML_SECRET in .env with the generated output.

Example:
```env
SAML_SECRET=<generated-value>
```
The secret should remain stable between deployments. Changing it invalidates existing SAML state

## 6) Configure Identity Provider metadata

The existing idp_metadata.xml file present in exerplaza/backend/saml_configuration/metadata/
is the working idp metadata used with the local keycloak idp. It will only work with said idp
so remove it if you are using your own configured idp.

Place the Identity Provider metadata file inside:

exerplaza/backend/saml_configuration/metadata/

The expected file is, so make sure it is named that way:

idp_metadata.xml

The metadata contains information required by the Service Provider to communicate with the Identity Provider, including:

entity identifier;
endpoints;
certificates.

For local testing, the simone-developement-keycloak branch contains the idp configuration used during development.
you can find it in the branch's exerplaza/backend/saml_configuration/metadata/

## 7) Verify Service Provider configuration

Ensure that:
```file
saml_sp_config.py
```
matches the configured Identity Provider.

The default configuration was tested with the Keycloak Identity Provider used during development.

Important values to verify:

ACS endpoint;
entity ID;
signing requirements;
certificate configuration;
metadata location.

refer to the pysaml2 documentation if you need to change any values in it.

## 7) Shut down the container

To complete the configuration you must either shut down the container or restart it

execute:
```bash
docker compose down
```
