# Setup

## Initial Setup

## 0) Make sure you pull the correct branch

At this moment the main branch does not have the saml implementation
so make sure the branch you pull does have the implementation
currently the simone-developement branch is the one that should be pulled as it has the updated full implementation.
make sure to git pull the branch.

Important: the current implementation of saml in exerplaza relies on Pysaml2, refer to the [PySAML2 documentation](https://pysaml2.readthedocs.io/) for any questions regarding the library.

## 1) Configure environment variables

Before starting the application, configure the SAML-related variables inside .env.
example:

```env
SAML_BASE_URL=http://127.0.0.1:54321
SAML_DEBUG=true
SAML_LOG_LEVEL=DEBUG
SAML_SECRET=0123456789 #temporary value
```

**Variable description**

| Variable | Purpose | Development value |
|---|---|---|
| `SAML_BASE_URL` | Public URL of the Service Provider. Must match the URL exposed by Flask and the URL configured in the IdP. | `http://127.0.0.1:54321` |
| `SAML_DEBUG` | Enables PySAML2 internal debugging logs. Useful during development, should generally be disabled in production. | `true` |
| `SAML_LOG_LEVEL` | Controls the verbosity of SAML-related logs. For production should be "INFO" | `DEBUG` |
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
- dependencies are installed;  
- the application environment exists;  
- required services are available.  

Be aware that the saml system at this stage is not yet configured and while you can reach it endpoints it
will not run even if you do.

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
- the certificate is readable;  
- the certificate has not expired;  
- the private key matches the certificate.  

The script will also tell you the location on generated files.

## 4) Extract generated files 

Extract generated files from next to certs.

This ensures the certificate/key pair is hooked up to the system.

## 5) Generate the SAML secret

Generate a cryptographically random secret:
```bash
docker compose exec -it web openssl rand -hex 32
```
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
```directory
exerplaza/backend/saml_configuration/metadata/
```
The expected file is, so make sure it is named that way:
```file
idp_metadata.xml
```
The metadata contains information required by the Service Provider to communicate with the Identity Provider, including:  
entity identifier;  
endpoints;  
certificates.  

## 7) Verify Service Provider configuration

Ensure that the file in:
```file
saml_sp_config.py
```
you can find it in:
```directory
exerplaza/backend/saml_configuration/
```
matches the configured Identity Provider.

The default configuration was tested with the Keycloak Identity Provider used during development.

Important values to verify:  
- ACS endpoint;  
- entity ID;  
- signing requirements;  
- certificate configuration;  
- metadata location.  

refer to the [PySAML2 config documentation](https://pysaml2.readthedocs.io/en/latest/howto/config.html) if you need to change any values in it.

## 8) Shut down the container

To complete the configuration you must either shut down the container or restart it

execute:
```bash
docker compose down
```
## Keycloak setup

## 1) Import docker-compose file with keycloak

from the simone-developement-keycloak branch import the docker-compose.yml file

## 2) rebuild the project

execute:
```bash
docker compose up --build
```

This will rebuild the project and automatically add the keycloak volume for data persistence and install all the needed tools to run keycloak locally. 
Once the installation is done wait until keycloak completes loading.

## 3) log in keycloak

go to:
```bash
http://localhost:8080/
```
and enter with the default credentials (you can change them after)
```bash
user: admin
pass: admin
```
## 4) configure keycloak

the default configuration is in the simone-developement-keycloak branch as:
```bash
http___127.0.0.1_54321_saml_sp.json
```
download it then export it once you've logged in keycloak

this will generate a realm and its related setting

optionally you can set your own settings, make sure to create an appropriate realm, correcly configure it with our exerplaza sp and create the correct client scopes and maps and make sure the sp is correcly configured to work with it.

## 5) export the keycloak metadata 

go to the keycloak metadata endpoint and copy the metadata

then go to the directory within exerplaza named:
```directory
exerplaza/backend/saml_configuration/metadata/
```
and copy it within the idp_metadata.xml file:
```file
idp_metadata.xml
```
remember to import the certificate for signing

## 6) begin idp testing

now that you've connected keycloak to exerplaza you can test

you can find a script to generate a test user in
```bash
exerplaza/backend/saml_configuration/scripts/
```
you can run it with:
```bash
docker compose exec -it web exerplaza/backend/saml_configuration/scripts/test_user.py
```
make sure to define a user with the same email in keycloak too by clicking users.

## changing settings

to change the exerplaza sp settings you must change the saml_sp_config you can find it in exerplaza/backend/saml_configuration make sure you follow the guidelines in [PySAML2 config documentation](https://pysaml2.readthedocs.io/en/latest/howto/config.html)

to apply any setting changes to exerplaza sp you must restart the project, do not keep the project running if you change any settings
