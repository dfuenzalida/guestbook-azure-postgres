# guestbook-azure-postgres

A version of the example [guestbook app from the Luminus framework](https://github.com/luminus-framework/examples/tree/master/guestbook), updated to use Postgres and ready to deploy to Azure in a few simple steps.

## Prerequisites

* Java 8 or Java 11 (Java 11 is used as the default in this project)
* [Leinigen](https://leiningen.org/)
* An Azure subscription ([free trial](https://azure.microsoft.com/en-us/free/))
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
* [Apache Maven](https://maven.apache.org/)

Optional, for local development

* Docker, to run PostgreSQL in a docker container
* GNU sed, to replace some entries in the `pom.xml` file before deploying. This is installed by default in most Linux distributions

## Differences with the original

* The `start-app` function from `core.clj` file has been updated to run the database migrations on the application startup, as explained in https://luminusweb.com/docs/deployment.html#heroku_deployment

* The migration file has been renamed to reflect that the file creates the `guestbook` table, instead of an `users` table.

* The table definition uses the syntax required for PostgreSQL instead of the original (H2). There are some minor differences on the definition of the primary key and timestamp.

## Local development


You can run a PostgreSQL instance with Docker:

```
docker run --name postgres -e POSTGRES_PASSWORD=pgpassword -d -p 5432:5432 postgres
```

Then connect to it from another container on the same network


```
docker run -it --rm --network host postgres psql -h localhost -U postgres
```

You can create a database just for the `guestbook` application and grant all permissions to a new user specific for the `guestbook` application.

```
postgres=# create database guestbook;
postgres=# create user guestbookuser with encrypted password 'guestbookpass';
postgres=# grant all privileges on database guestbook to guestbookuser;
```

The original example used SQL for an in-memory H2 database. In postgreSQL the table migration file has been update to:

```sql
CREATE TABLE guestbook
(id SERIAL PRIMARY KEY,
 name VARCHAR(30),
 message VARCHAR(200),
 timestamp TIMESTAMP);
```

## Deployment on Azure

### Creating cloud resources

This step needs to be run only once to create the required resources in Azure.

If you haven't logged before, login into your Azure account with `az login` and follow the prompts.

Edit the script `create-resources.sh` provided. The beginning of the file contains a configuration section where you can adjust some parameters such as the deployment region, and the database username and password.

The script will create the following resources for you:

* A resource group which will contain every other resource for this application
* A server to run a PostgreSQL instance
* A database in the PostgreSQL host
* A configuration entry to allow connections from services hosted in Azure to the database server
* An Application Service plan that can deploy Linux hosts
* A definition for the guestbook web application
* A configuration entry with the JDBC URL to connect from the guestbook web app to the database 

Run the script with

```
./create-resources.sh
```

Once all the resources are created correctly, it should print `All Azure resources created` in the console.

### Configure the deployment

Generate a Maven `pom.xml` file by running `lein pom`:

```
$ lein pom
Wrote .../guestbook/pom.xml
```

Run the following command line to use a Maven plugin which will detect and configure most of the parameters required for deployment:

```
mvn com.microsoft.azure:azure-webapp-maven-plugin:2.0.0:config
```

The command above will prompt you to create a new Web App or to use an existing one. Choose the option `2` to use the existing Web App that you created in the previous step using the script:

```
$ mvn com.microsoft.azure:azure-webapp-maven-plugin:2.0.0:config
[INFO] Scanning for projects...
...
[INFO] Auth Type : AZURE_CLI, Auth Files : [/home/user/.azure/azureProfile.json, /home/user/.azure/accessTokens.json]
...
[INFO] It may take a few minutes to load all Java Web Apps, please be patient.
Java SE Web Apps in subscription My Subscription:
* 1: <create>
  2: guestbook-25026 (linux, java 11-java11)
Please choose a Java SE Web App [<create>]: 2 <== CHOOSE TO USE EXISTING APP
Please confirm webapp properties
Subscription Id : (...redacted...)
AppName : guestbook-25026
ResourceGroup : guestbook-25026-rg
Region : westus2
PricingTier : Free_F1
OS : Linux
Java : Java 11
Web server stack: Java SE
Deploy to slot : false
Confirm (Y/N) [Y]: Y <== CONFIRM THAT YOU WANT TO USE THE CURRENT SETTINGS
[INFO] Saving configuration to pom.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  19.405 s
[INFO] Finished at: 2021-02-28T14:18:36-08:00
[INFO] ------------------------------------------------------------------------
```

### Update the POM file

The file `pom.xml` generated by the `azure-webapp-maven-plugin` needs a minor tweak for deploying an Uberjar package.

You will need to edit the file `pom.xml` and replace the values in the following section:

```xml
  <deployment>
    <resources>
      <resource>
        <directory>${project.basedir}/target/uberjar</directory><!-- new path -->
        <includes>
          <include>guestbook.jar</include><!-- single file -->
        </includes>
      </resource>
    </resources>
  </deployment>
```

You can perform the text replacements from the command line using `sed`, like the following:

```
sed -i 's/\/target/\/target\/uberjar/' pom.xml
sed -i 's/\*\.jar/guestbook\.jar/' pom.xml
```

Now, generate the Uberjar file to be deployed with `lein uberjar`:

```
$ lein uberjar
Compiling guestbook.config
Compiling guestbook.core
Compiling guestbook.db.core
Compiling guestbook.env
Compiling guestbook.handler
Compiling guestbook.layout
Compiling guestbook.middleware
Compiling guestbook.middleware.formats
Compiling guestbook.nrepl
Compiling guestbook.routes.home
Created /home/user/Projects/azure/guestbook/target/uberjar/guestbook-0.1.0-SNAPSHOT.jar
Created /home/user/Projects/azure/guestbook/target/uberjar/guestbook.jar
```

Finally, you can deploy with `mvn azure-webapp:deploy`:

```
$ mvn azure-webapp:deploy
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------------< guestbook:guestbook >-------------------------
[INFO] Building guestbook 0.1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- azure-webapp-maven-plugin:2.0.0:deploy (default-cli) @ guestbook ---
...
[INFO] Trying to deploy artifact to guestbook-25026...
[INFO] Deploying (/.../guestbook/target/uberjar/guestbook.jar)[jar]  ...
[INFO] Successfully deployed the artifact to https://guestbook-25026.azurewebsites.net
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  27.655 s
[INFO] Finished at: 2021-10-13T23:49:22-07:00
[INFO] ------------------------------------------------------------------------
```

Give it a minute for the application to deploy and warm up. The first time you visit the application you may see an error 500 because the migrations might have not completed, but after that it should work fine.

If you want to delete the application and all the related resources, go to https://portal.azure.com/#blade/HubsExtension/BrowseResourceGroups then select the appropriate resource group and delete it.

## License

Copyright © 2021 Denis Fuenzalida
