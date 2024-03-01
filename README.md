# qdtDevelopment

This repo contains various artifacts to setup the webMethods API development environment using Docker Compose

##  Content of the Docker Compose stack

### Postgres database

We create a sandbox database, which we access using the JDBC adapter.
This database could also be used for the ISInternal, ISCoreAudit, ISDashboardStats and other webMethods internal schemas, but I've kept things simple here and configured an embedded database.

The container is initialized with a user and a password that are defined in the environment.variables file.

### Universal Messaging

We create a simple UM realm to deal with the messaging aspects (usually publishing and consumption of EDA-style events.)
We use the product image coming from https://containers.softwareag.com here.
The UM realm needs a license file that we inject using a docker volume.

### Microservice Runtime

Finally, we also set up a MSR container, this time using a custom image that I have built. For integrations the product images in https://containers.softwareag.com are usually not sufficient, you need to add packages and other dependencies in order to have a usable platform. It's a little bit like Node.js, you usually needs to add external packages to deal with your implementation.  
The process to create the custom base image is described in the qdtBase repository: https://github.com/staillansag/qdtBase.git  

Similarly to the UM, the MSR needs a product license and we inject it the same way, using a docker volume.  

The MSR is configured using an application.properties file that is also injected into the container using a volume. Part of the properties referenced in it point to environment variables, which are injected using the environment.variables file.

Last but not least, the integration packages are also mounted using volumes, 1 volume per package. This allows local access to the packages, even though they are deployed in the containerized MSR. We can then easily use git clients to deal with version control on these packages.

##  Installation

Follow this process, step by step.

1.  Clone this repository

Simply use this command:
```git clone https://github.com/staillansag/qdtDevelopment.git```

2.  Manage the configuration

Rename the environment.variables.example file into environment.variables, and choose passwords for Postgres and the MSR Administrator user.  

For Salesforce connectivity, this will be managed later.

Note that we don't need to change the application.properties file.  


3.  Manage the MSR license

Create a license folder and please your MSR license in it, renaming it to msr-license.xml  

4.  Manage the integration packages

Create a packages folder, and cd into it. Then clone the github repos using the following commands:
```
git clone https://github.com/staillansag/qdtReferenceData.git
git clone https://github.com/staillansag/qdtContactManagement.git
git clone https://github.com/staillansag/qdtBackendForFrontend.git
git clone https://github.com/staillansag/qdtAccountManagement.git
git clone https://github.com/staillansag/qdtFramework.git
```

5.  Start the Docker Compose stack

Go back to the root qdtDevelopment folder and issue the following command:
```
docker-compose up -d
```

In some environments, it's going to be:
```
docker compose up -d
```

In some environments I have also seen issues with the health checks configured for the MSR container.
If that's your case, then remove these lines from the docker-compose.yml file:

```
    healthcheck:
      interval: 5s
      retries: 24
      test: ["CMD-SHELL", "curl -o /dev/null -s -w '%{http_code}' http://localhost:5555 | grep -qE '^(200|3[0-9]{2})$'"]
```

The output of the Docker compose command is also environment specific, here's what I see:
```
[+] Running 4/4
 ✔ Network qdt_sag   Created                                                                                                                                                                                                         0.0s
 ✔ Container postgresql  Started                                                                                                                                                                                                         0.0s
 ✔ Container umserver    Started                                                                                                                                                                                                         0.0s
 ✔ Container msr         Started
 ```

 The Postgres and UM containers should only need a few seconds to start, but the MSR container will need a bit more time, probably between 30 and 120 seconds depending on your hardward config.  

 Try connecting to the MSR admin console at http://localhost:5555 with the Administrator user and the password you defined in the environment.variables file.  
 Check the following:
 -  Messaging > JMS Settings > DEFAULT_IS_JMS_CONNECTION is enabled
 -  Adapters > webMethods Adapter for JDBC > Connections: the two connections are enabled
 -  Packages > Management: the 5 mounted qdt packages are visible and enabled

If you need to do some troubleshooting, then use the following command, which displays the server.log of the MSR:
```
docker logs msr
```

6.  Connect the designer to the MSR

Open you designer and go to Preferences (or Settings in MacOS), Software AG submenu, Integration Server, and then add a new server which points to your containerized MSR:
-   Name: Docker (or anything you like)
-   Host: localhost
-   Port: 5555
-   User: Administrator
-   Password: the password you defined for the Administrator user in environment.variables

You should then be able to connect to the containerized MSR and see its packages.  

7.  Finish the CloudStreams Salesforce config

TODO

8.  Create the Postgres tables

There are DDL files in the following locations:
-   ./packages/qdtContactManagement/resources/database/contacts.ddl.sql
-   ./packages/qdtContactManagement/resources/database/contacts-roles.ddl.sql
-   ./packages/qdtReferenceData/resources/database/reference_data.ddl.sql

There is also this batch of SQL insert commands:
-   ./packages/qdtReferenceData/resources/database/reference_data.insert.sql

These SQL files can be executed using any Database editor that's connected to Postgres.  
Alternatively you can execute them using the following commands:
```
docker cp ./packages/qdtContactManagement/resources/database/contacts.ddl.sql postgresql:/tmp/contacts.ddl.sql
docker cp ./packages/qdtContactManagement/resources/database/contacts-roles.ddl.sql postgresql:/tmp/contacts-roles.ddl.sql
docker cp ./packages/qdtReferenceData/resources/database/reference_data.ddl.sql postgresql:/tmp/reference_data.ddl.sql
docker cp ./packages/qdtReferenceData/resources/database/reference_data.insert.sql postgresql:/tmp/reference_data.insert.sql
docker exec -it postgresql psql -U postgres -d postgres -f /tmp/contacts.ddl.sql
docker exec -it postgresql psql -U postgres -d postgres -f /tmp/contacts-roles.ddl.sql
docker exec -it postgresql psql -U postgres -d postgres -f /tmp/reference_data.ddl.sql
docker exec -it postgresql psql -U postgres -d postgres -f /tmp/reference_data.insert.sql
```

9.  Import the postman assets

There are three Postman collections, 1 for each API:
./packages/qdtContactManagement/resources/tests/ContactManagementAutomated.postman_collection.json
./packages/qdtAccountManagement/resources/tests/AccountManagement.postman_collection.json
./packages/qdtBackendForFrontend/resources/tests/BackendForFrontend.postman_collection.json

And same for the environments:
./packages/qdtContactManagement/resources/tests/ContactManagement.postman_environment.json
./packages/qdtAccountManagement/resources/tests/AccountManagement.postman_environment.json
./packages/qdtBackendForFrontend/resources/tests/BackendForFrontend.postman_environment.json

Import these 6 files into Postman.

For the environments, do the following adjustments:
-   For AccountManagement, the url is http://localhost:5555/rad/qdtAccountManagement.api:AccountManagementAPI
-   For ContactManagement, the url is http://localhost:5555/rad/qdtContactManagement.api:ContactManagementAPI
-   For BackendForFrontEnd, the url is http://localhost:5555/rad/qdtBackendForFrontend.apiServer:BackendForFrontendAPI_1_0_0
-   For the three environment:
    -   userName = Administrator
    -   password = the password you defined for the Administrator user in environment.variables

Now you're ready to test the APIs with Postman.

