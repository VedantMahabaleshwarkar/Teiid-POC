# Teiid-POC

The goal of this POC is to demonstrate the use of the [Teiid Operator](https://github.com/teiid/teiid-operator) to do the following

1. Use Teiid as a gating layer between multiple data sources.
   1. Postgresql
   2. Rook Ceph
   3. AWS s3
   4. AWS RedShift
2. Set up access control policies for reads and writes to the data.
   1. Use the KeyCloak Operator to manage access control
   
## Overview
![image](https://github.com/VedantMahabaleshwarkar/Teiid-POC/blob/master/Teiid%20Poc.png)  

## Access Control  
![image](https://github.com/VedantMahabaleshwarkar/Teiid-POC/blob/master/access%20control%20teiid.png)

## Getting Started

The first step is to install the Teiid Operator from OLM.
As an alternative, a guide to manually build and deploy the Teiid Operator can be found [here](https://github.com/teiid/teiid-operator#development-of-operator)

### Setting up an example

In this example we will setup a Postgresql Datasource, populate it with some data and add the postgres instance to the VDB.

#### Setting up the data

````
oc new-app -f postgresql-ephemeral-template.json \
  -p DATABASE_SERVICE_NAME=accounts \
  -p POSTGRESQL_USER=user \
  -p POSTGRESQL_PASSWORD=changeit \
  -p POSTGRESQL_DATABASE=accounts
````
Note the username and password.

To populate the instance with data, rsh into the postgres pod and execute
````
oc rsync . accounts-xxxxx:/tmp
psql -U user -d accounts -f /tmp/accounts-schema.sql
````

#### Building the Virtual Database (VDB)

The VDB is defined by a yaml file of kind `VirtualDatabase` which is a CRD created by the Operator.
The following chunk of the file shows how to add datasources to the VDB.
````
apiVersion: teiid.io/v1alpha1
kind: VirtualDatabase
metadata:
  name: portfolio
spec:
  replicas: 1
  datasources:
    - name: accountdb
      type: postgresql
      properties:
        - name: username
          value: user
        - name: password
          value: changeit
        - name: jdbc-url
          value: jdbc:postgresql://accounts/accounts
    - name: quotesvc
      type: rest
      properties:
        - name: endpoint
          value: https://finnhub.io/api/v1/
````
Note the password and username for the postgres source, it should match with the values used earlier.

The creation of the VDB is defined by DDL as follows.
````
apiVersion: teiid.io/v1alpha1
kind: VirtualDatabase
metadata:
  name: portfolio
spec:
  replicas: 1
    expose:
    - LoadBalancer
  datasources:
    - name: accountdb
      type: postgresql
      properties:
        - name: username
          value: user
        - name: password
          value: password
        - name: jdbc-url
          value: jdbc:postgresql://accounts/accounts
    - name: quotesvc
      type: rest
      properties:
        - name: endpoint
          value: https://finnhub.io/api/v1/
  build:
    source:
      ddl: |
        CREATE DATABASE Portfolio OPTIONS (ANNOTATION 'The Portfolio VDB');
        USE DATABASE Portfolio;

        --############ translators ############
        CREATE FOREIGN DATA WRAPPER rest;
        CREATE FOREIGN DATA WRAPPER postgresql;

        --############ Servers ############
        CREATE SERVER "accountdb" FOREIGN DATA WRAPPER postgresql;
        CREATE SERVER "quotesvc" FOREIGN DATA WRAPPER rest;

        --############ Schemas ############
        CREATE SCHEMA marketdata SERVER "quotesvc";
        CREATE SCHEMA accounts SERVER "accountdb";

        CREATE VIRTUAL SCHEMA Portfolio;

        --############ Schema:marketdata ############
        SET SCHEMA marketdata;

        IMPORT FROM SERVER "quotesvc" INTO marketdata;

        --############ Schema:accounts ############
        SET SCHEMA accounts;

        IMPORT FROM SERVER "accountdb" INTO accounts OPTIONS (
                "importer.useFullSchemaName" 'false',
                "importer.tableTypes" 'TABLE,VIEW');

        --############ Schema:Portfolio ############
        SET SCHEMA Portfolio;

        CREATE VIEW StockPrice (
            symbol string,
            price double,
            CONSTRAINT ACS ACCESSPATTERN (symbol)
        ) AS
            SELECT p.symbol, y.price
            FROM accounts.PRODUCT as p, TABLE(call invokeHttp(action=>'GET', endpoint=>QUERYSTRING('quote', p.symbol as "symbol", 'bq0bisvrh5rddd65fs70' as "token"), headers=>jsonObject('application/json' as "Content-Type"))) as x,
            JSONTABLE(JSONPARSE(x.result,true), '$' COLUMNS price double path '@.c') as y

        CREATE VIEW AccountValues (
            LastName string PRIMARY KEY,
            FirstName string,
            StockValue double
        ) AS
            SELECT c.lastname as LastName, c.firstname as FirstName, sum((h.shares_count*sp.price)) as StockValue
            FROM Customer c JOIN Account a on c.SSN=a.SSN
            JOIN Holdings h on a.account_id = h.account_id
            JOIN product p on h.product_id=p.id
            JOIN StockPrice sp on sp.symbol = p.symbol
            WHERE a.type='Active'
            GROUP BY c.lastname, c.firstname;
````
The above code can be saved as `portfolio.yaml` and can be deployed by using
````
oc apply -f portfolio.yaml
````
To check success of deployment, check for `phase:Running` in the following command
````
oc get vdb portfolio -o yaml | grep phase
````

#### Accessing the VDB

The VDB can be accessed using a postgres client or Apache Superset or by creating an sqlline pod in the cluster.

##### Apache Superset Connection

The superset connection string looks like
`postgresql+psycopg2://user:password@host:35432/vdb`
`/vdb` should be replaced by the name of your VDB

Currently there is a bug in the Operator that flips the `targetPort` and `port` in the `Service` yaml for the VDB. Please ensure that `targetPort` is `5432` and `port` is `35432` for the `pg` Port.

Test the connection in Superset and the VDB should be ready to be queried using Superset.

##### Using sqlline

To quickly test a VDB, execute the following
````
oc run -it --restart=Never --attach --rm --image quay.io/asmigala/sqlline:latest sqlline
````
This will open up a prompt that looks like this
````
sqlline>
````

Execute the following
````
sqlline> !connect jdbc:teiid:portfolio@mm://portfolio:31000;

Enter username for jdbc:teiid:portfolio@mm://portfolio:31000;: foo
Enter password for jdbc:teiid:portfolio@mm://portfolio:31000;: ****

0: jdbc:teiid:portfolio@mm://portfolio:31000>
````

You can enter following general commands here to help with tool
````
!dbinfo
!tables
!help
!quit
````

### Adding Data sources

#### Rook Ceph

The first step would be to create your own Rook Ceph instance

[Instructions to create Rook Cluster](https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md#tldr)

[Instructions to create Rook Object Store](https://github.com/rook/rook/blob/master/Documentation/ceph-object.md)

The Rook Ceph instance can be added as a datasource according to the syntax defined [here](https://github.com/teiid/teiid-openshift-examples/blob/master/datasources.adoc#configuring-cephminiorook-as-source-using-s3)

#### Aws s3

Create an s3 bucket and populate it with data.

The s3 bucket can be added as a datasource according to syntax defined [here](https://github.com/teiid/teiid-openshift-examples/blob/master/datasources.adoc#configuring-amazon-s3-as-source)

#### Aws RedShift

Note: This has not been tested yet

Documentation can be found [here](https://github.com/teiid/teiid-openshift-examples/blob/master/datasources.adoc#configuring-relational-database)

### Defining the DDL for addition Datasource

For each of the above DataSources, the DDL statements have to be written according to the `translators` for that source.
The Documentation for can be found [here](http://teiid.github.io/teiid-documents/13.0.x/content/reference/Translators.html)

### Configuring Security

Firstly, Install the KeyCloak Operator from OLM.
Then create the KeyCloak instance by running the following
````
oc create -f files/keycloak.yaml
````
This will create a KeyCloak instance with the name `dv-keycloak`

Create a Realm in this KeyCloak instance by running
````
oc create -f files/realm.yaml
````

Now create the KeyCloak client
In the file `files/client.yaml` you will need to edit the `Spec.Client.redirectUris` field.
The correct value for this field can be obtained as follows
````
oc get vdb portfolio -o yaml | grep route
````
This should give a route, from the resulting route, omit the `/odata` part and copy the rest in the `client.yaml` file

Finally,
````
oc create -f files/client.yaml
````

Now, we will create a user in the KeyCloak realm that can access the VDB
A sample user yaml for user `John` is provided
````
oc create -f files/user.yaml
````
Delete the previous VDB and apply the new one by running
````
oc create -f files/portfolio.yaml
````

In the new VDB we add the following section to the yaml
````
env:
  - name: KEYCLOAK_REALM
    value: dv
  - name: KEYCLOAK_RESOURCE
    value: portfolio
  - name: KEYCLOAK_CREDENTIALS_SECRET
    value: changeit
  - name: KEYCLOAK_AUTH_SERVER_URL
    value: https://keycloak-datavirt.apps.vmahabal.dev.datahub.redhat.com/auth <=== ***** CHANGE THIS
  - name: KEYCLOAK_DISABLE_TRUST_MANAGER
    value: "true"
````
The `KEYCLOAK_AUTH_SERVER_URL` field has to be changed and the correct value can be found by running
````
oc get Keycloak dv-keycloak -o yaml | grep internalURL:
````
This will output a route, add `/auth` to it and paste the resulting string into the yaml.

Once the VDB is running it can be tested by the Sqlline pod as follows
````
oc run -it --restart=Never --attach --rm --image quay.io/asmigala/sqlline:latest sqlline
````
````
sqlline> !connect jdbc:teiid:portfolio@mm://portfolio:31000;

Enter username for jdbc:teiid:portfolio@mm://portfolio:31000;: john
Enter password for jdbc:teiid:portfolio@mm://portfolio:31000;: changeit

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From CustomerHoldings where lastname='Doe'
````

### Configuring Access control

Now that we have configured security, we can give `Roles` to users and add access control.

Firstly, access the KeyCloak Admin UI through the appropriate route.
The login credentials for the Admin UI can be found in the `credential-dv-keycloak` secret.
Switch to the DV Realm that we created earlier.
Create 2 Roles in the Admin UI, `account-owner` and `broker`

Create the following users, but delete the `user.yaml` we created earlier
`oc create -f files/john.yaml`
`oc create -f files/bob.yaml`

We have given John the Role of `account-owner` and Bob has been given the role of `broker`.

Now we can re-create our portfolio to apply access control according to these Roles.
Delete the `portfolio.yaml` created earlier and then run the following
`oc create -f files/portfolio.yaml`

Basically, we have added this to the vdb
````
GRANT SELECT ON TABLE portfolio.StockPrice TO "all";
GRANT SELECT ON TABLE portfolio.AccountValues TO "broker";
GRANT SELECT ON TABLE portfolio.CustomerHoldings TO "broker";
GRANT SELECT ON TABLE portfolio.CustomerHoldings TO "account-owner";
CREATE POLICY policyViewOwn ON portfolio.CustomerHoldings TO "account-owner" USING (FirstName = user(false));
````
These lines configure the access control for tables according to the Roles defined in our KeyCloak Realm.


Run the following sets of commands to test
````
sqlline> !connect jdbc:teiid:portfolio@mm://portfolio:31000;

Enter username for jdbc:teiid:portfolio@mm://portfolio:31000;: john
Enter password for jdbc:teiid:portfolio@mm://portfolio:31000;: changeit

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From CustomerHoldings WHERE Lastname='Doe';

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From AccountValues;

0: jdbc:teiid:portfolio@mm://portfolio:31000> !quit
````

````
sqlline> !connect jdbc:teiid:portfolio@mm://portfolio:31000;

Enter username for jdbc:teiid:portfolio@mm://portfolio:31000;: bob
Enter password for jdbc:teiid:portfolio@mm://portfolio:31000;: changeit

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From CustomerHoldings WHERE Lastname='Smith';

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From CustomerHoldings WHERE Lastname='Doe';

0: jdbc:teiid:portfolio@mm://portfolio:31000> SELECT * From AccountValues;

0: jdbc:teiid:portfolio@mm://portfolio:31000> !quit
````
