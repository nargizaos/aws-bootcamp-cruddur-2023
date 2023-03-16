# Week 4 — Postgres and RDS

### Provision RDS Instance

We will createe RDS instance from AWS CLI

```
aws rds create-db-instance \
  --db-instance-identifier cruddur-db-instance \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version  14.6 \
  --master-username cruddurroot \
  --master-user-password  yourpassword \
  --allocated-storage 20 \
  --availability-zone us-east-1a \
  --backup-retention-period 0 \
  --port 5432 \
  --no-multi-az \
  --db-name cruddur \
  --storage-type gp2 \
  --publicly-accessible \
  --storage-encrypted \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --no-deletion-protection
```
Here we are using ```db.t3.micro``` instance type, Postgres default version ```14.6```, storage type ```gp2``` because it's free tier.
When your RDS Instance comes up active in Console, stop it in order to save money on spending, because RDS is not free tier. 

<img src="https://user-images.githubusercontent.com/66444859/225451906-ec2742a0-2c6a-45b1-816e-762c5c2e03b1.png" width=65%>

In prevous classes we have added db instance code in our ```docker-compose.yaml```:
```
db:
    image: postgres:13-alpine
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - '5432:5432'
    volumes: 
      - db:/var/lib/postgresql/data
```

Start Postgres instance up  by running ```Compose Up``` to connect to RDS instance.

Connect to PostgreSQL:
```psql -Upostgres --host localhost```

Here are some common PSQL commands:
```
\x on -- expanded display when looking at data
\q -- Quit PSQL
\l -- List all databases
\c database_name -- Connect to a specific database
\dt -- List all tables in the current database
\d table_name -- Describe a specific table
\du -- List all users and their roles
\dn -- List all schemas in the current database
CREATE DATABASE database_name; -- Create a new database
DROP DATABASE database_name; -- Delete a database
CREATE TABLE table_name (column1 datatype1, column2 datatype2, ...); -- Create a new table
DROP TABLE table_name; -- Delete a table
SELECT column1, column2, ... FROM table_name WHERE condition; -- Select data from a table
INSERT INTO table_name (column1, column2, ...) VALUES (value1, value2, ...); -- Insert data into a table
UPDATE table_name SET column1 = value1, column2 = value2, ... WHERE condition; -- Update data in a table
DELETE FROM table_name WHERE condition; -- Delete data from a table
```

When connected to PostgreSQL, run ```\l``` to get a list of databases.

<img src="https://user-images.githubusercontent.com/66444859/224901244-40cbdf49-0877-4d93-b74f-244341521bc8.png" width=45%>

### Create (and dropping) our database

We can use the createdb command to create our database:
https://www.postgresql.org/docs/current/app-createdb.html

```createdb cruddur -h localhost -U postgres```
```psql -U postgres -h localhost```
```
\l
DROP database cruddur;
```

All of the above commands are alias to this command which we will use to create the database within the PSQL client
```CREATE database cruddur;```

### Import Script

We'll create a new SQL file called schema.sql and we'll place it in backend-flask/db. It will contain the schema/the structure of our database.
We are going to keep writing things in this file and load/unload it. 

The command to import:
```psql cruddur < db/schema.sql -h localhost -U postgres```

### Add UUID Extension

Before importing script we will add UUID extensions. UUID stands for Universally Unique Identifier. It will generate random string which will be attached to IDs or names of the tables.

We are going to have Postgres generate out UUIDs. We'll need to use an extension called:

```
CREATE EXTENSION "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```
We will put only the second one which will add only if extension does not exists. 

If don't want to type password every time we connect to Postgres, we can create a ```CONNECTION_URL``` which will contain the endpoint nad password. 
Here is the ```CONNECTION_URL``` command to run from CLI: 

```
export CONNECTION_URL="postgresql://postgres:password@localhost:5432/cruddur"
gp env CONNECTION_URL="postgresql://postgres:password@127.0.0.1:5432/cruddur"
```
Now we can connect with this short command:

```psql $CONNECTION _URL```

Connection URL for Prod: 

```
export PROD_CONNECTION_URL="postgresql://cruddurroot:yourrdspassword@cruddur-db-instance.czuqvdoungei.us-east-1.rds.amazonaws.com:5432/cruddur"
gp env PROD_CONNECTION_URL="postgresql://cruddurroot:yourrdspassword@cruddur-db-instance.czuqvdoungei.us-east-1.rds.amazonaws.com:5432/cruddur"
```

### Shell script to drop the database

For things we commonly need to do we can create a new directory called ```bin``` in ```backend-flask``` folder.

We'll create an new folder called bin to hold all our bash scripts.

```
mkdir /workspace/aws-bootcamp-cruddur-2023/backend-flask/bin
```

Create the files on ```bin``` folder
```
db-create
db-drop
db-schema-load
```

To drop database we can put this command in ```db-drop```:

```
#! /usr/bin/bash
psql $CONNECTION _URL -c "drop database cruddur;" 
```

In order to execute bash script we need to make these files executable. We'' use ```chmod``` command for that.

```
ls -l ./bin
chmod u+x bin/db-create
chmod u+x bin/db-drop
chmod u+x bin/db-schema-load 
```

Try executing drop script:
```
./bin/db-drop
```

We ran into error  ```cannot drop currently open database```.  When we are using ```CONNECTION_URL``` we are connecting to database and trying to drop it while being connected to it. Our ```CONNECTION_URL``` should exclude ```database``` part. We'll use ```sed``` command to exlude database from ```CONNECTION_URL```.  

```sed 's/#select/#replacewith/g'```

Replace command in ```db-drop``` to this command:

```NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<<"$CONNECTION_URL")```

```
./bin/db-drop
```

### Shell Script to Connect to DB

In ```db-create``` file:

```
#! /usr/bin/bash
echo "db-create"

NO_DB_CONNECTION_URL=$(sed 's/\/cruddur//g' <<< "$CONNECTION_URL")
psql $NO_DB_CONNECTION_URL -c "create database cruddur;"

```

```
./bin/db-create
```

In ```db-schema-load```:

```
#! /usr/bin/bash

echo "db-schema-load"

psql $CONNECTION_URL cruddur < db/schema.sql
```

```
./bin/db-schema-load`
```

This execution command only works while we are inside ```backend-flask``` folder. To make this work outside of that folder we'll use ```realpath``` feature. 

```
#! /usr/bin/bash

echo "db-schema-load"

schema_path="$(realpath .)/db/schema.sql"
echo $schema_path

psql $CONNECTION_URL cruddur < $schema_path
```
```
./backend-flask/bin/db-schema-load`
```
It's not working for now, but we'll come back to it later. 


We need to find a way to toggle between our ```local``` and ```Prod``` mode. We can do ```if else``` statement for that. 






