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

