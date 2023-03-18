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


<img src="https://user-images.githubusercontent.com/66444859/226124317-4bd65d29-57ad-40a2-bcc2-86c173b4cb91.png" width=50%>


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

<img src="https://user-images.githubusercontent.com/66444859/226124360-954df8a0-4f7f-47a8-ac36-3151130698af.png" width=65%>


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
./backend-flask/bin/db-schema-load
```
It's not working for now, but we'll come back to it later. 


We need to find a way to toggle between our ```local``` and ```Prod``` mode. We can do ```if else``` statement for that. 

In ```db-schema-load```:

```
if ["$1" = "prod"]; then
  echo "using production key"
  CON_URL=$PROD_CONNECTION_URL
else
  CON_URL=$CONNECTION_URL
fi
```

Run:
```
./bin/db-schema-load prod
```

It should show ```using production``` and it's just hanging there because we stopped our rds instance from console. 


### Create our tables
https://www.postgresql.org/docs/current/sql-createtable.html

We will be creating ```user``` table and ```activities``` table. 

In ```db/schema-sql```:

```
CREATE TABLE public.users (
  uuid UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  user_uuid UUID NOT NULL,
  display_name text,
  handle text,
  cognito_user_id text,
  created_at TIMESTAMP default current_timestamp NOT NULL
);
```

```
CREATE TABLE public.activities (
  uuid UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  message text NOT NULL,
  replies_count integer DEFAULT 0,
  reposts_count integer DEFAULT 0,
  likes_count integer DEFAULT 0,
  reply_to_activity_uuid integer,
  expires_at TIMESTAMP,
  created_at TIMESTAMP default current_timestamp NOT NULL
);
```
We want to delete tables before creating, so we need to put it above ```CREATE TABLE``` :

```
DROP TABLE IF EXISTS public.users;
DROP TABLE IF EXISTS public.activities;
```
Create tables:
```
./bin/db-schema-load
```

### Shell Script to Connect to DB

We need to create script to connect. Create ```db-connect``` in ```bin```.

```
export CONNECTION_URL="postgresql://postgres:pssword@127.0.0.1:5433/cruddur"
gp env CONNECTION_URL="postgresql://postgres:pssword@127.0.0.1:5433/cruddur"
```
In ```bin/db-connect```:

```
#! /usr/bin/bash

psql $CONNECTION_URL
```

Make file executable:

```
chmod u+x /bin/db-connect
```
Execute the script:

```./bin/db-connect```

 List all tables in the current database
 
```\dt```

### Shell script to load the seed data

Create ```db-seed``` in ```bin```.

```
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-seed"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

seed_path="$(realpath .)/db/seed.sql"
echo $seed_path

if ["$1" = "prod"]; then
  echo "using production key"
  CON_URL=$PROD_CONNECTION_URL
else
  CON_URL=$CONNECTION_URL
fi

psql $CON_URL cruddur < $seed_path
```


Make file executable:

```
chmod u+x /bin/db-seed
```

Create ```seed.sql``` file in ```db``` folder. We need it to give initial values, insert data rows.

```
-- this file was manually created
INSERT INTO public.users (display_name, handle, cognito_user_id)
VALUES
  ('Andrew Brown', 'andrewbrown' ,'MOCK'),
  ('Andrew Bayko', 'bayko' ,'MOCK');

INSERT INTO public.activities (user_uuid, message, expires_at)
VALUES
  (
    (SELECT uuid from public.users WHERE users.handle = 'andrewbrown' LIMIT 1),
    'This was imported as seed data!',
    current_timestamp + interval '10 day'
  )
```

Execute the ```db-seed``` script:

```./bin/db-seed```

We ran into ```user_uuid does not exist```. We forgot to insert ```user_uuid UUID NOT NULL,``` in ```schema-sql```.

After adding, run ```schema-load```:

```./bin/db-schema-load```

<img src="https://user-images.githubusercontent.com/66444859/226056253-7aae415e-a8e1-4142-8f44-216f885c660d.png" width=60%>

Seed our data:

```./bin/db-seed```

<img src="https://user-images.githubusercontent.com/66444859/226055737-d418f2ca-528b-4a98-9ab9-e3e887e207b8.png" width=60%>

Connect to Postgres with ```./bin/db-connect
Run this command to see seed from ```activities```:

```
SELECT * FROM activities;
```
<img src="https://user-images.githubusercontent.com/66444859/226063484-26190b75-bc5d-4eb1-9048-2abd8fcdebe6.png" width=50%>


### Make prints nicer
We we can make prints for our shell scripts coloured so we can see what we're doing:

https://stackoverflow.com/questions/5947742/how-to-change-the-output-color-of-echo-in-linux

```
CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-schema-load"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"
```

### See what connections we are using

When we try to drop the database, we get an error that we have 3 other sessions.

<img src="https://user-images.githubusercontent.com/66444859/226064255-31009c9d-f14d-4b5d-be9f-ed0945344ece.png" width=50%>

We need to find out what are those 3 active connections we have. 
Create new file ```/bin/db-sessions``` and pase this script:

```
#! /usr/bin/bash

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-sessions"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n" 

if ["$1" = "prod"]; then
  echo "using production key"
  URL=$PROD_CONNECTION_URL
else
  URL=$CONNECTION_URL
fi

NO_DB_URL=$(sed 's/\/cruddur//g' <<<"$URL")
psql $NO_DB_URL -c "select pid as process_id, \
       usename as user,  \
       datname as db, \
       client_addr, \
       application_name as app,\
       state \
from pg_stat_activity;"
```
Make file executable

```
chmod u+x ./bin/db-sessions
```

Run ```./bin/db-sessions```

<img src="https://user-images.githubusercontent.com/66444859/226065073-3dc4d867-b457-4257-8900-f8650412be68.png" width=50%>

We had tried to close session by closing the connection we opened through GitPod Database Explorer, but sessions were still there. Tried to bring down everything by Compose Down, bring up with Compose Up and sessions are gone. 

Create new file ```db-setup``` in ```bin```. 

```
#! /usr/bin/bash
-e # stop if it fails at any point

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-setup"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

bin_path="$(realpath .)/bin"

source "$bin_path/db-drop"
source "$bin_path/db-create"
source "$bin_path/db-schema-load"
source "$bin_path/db-seed"
```
Make file executable

```
chmod u+x ./bin/db-setup
```

Run ```./bin/db-setup``` and we see that everything was executed, db, table created, seed was inserted.

<img src="https://user-images.githubusercontent.com/66444859/226067699-cd473330-d297-43e1-8fb1-dc73801129eb.png" width=50%>

### Install Postgres Driver in Backend Application

We have to get Driver installed for Postgres. We will install PostgreSQL driver for Python - Psycopg. 
https://www.psycopg.org/psycopg3/

We'll add the following to our ```requirments.txt```

```
psycopg[binary]
psycopg[pool]
```

```
pip install -r requirements.txt
```

```db.py```

```
def query_wrap_object(template):
  sql = '''
  (SELECT COALESCE(row_to_json(object_row),'{}'::json) FROM (
  {template}
  ) object_row);
  '''

def query_wrap_array(template):
  sql = '''
  (SELECT COALESCE(array_to_json(array_agg(row_to_json(array_row))),'[]'::json) FROM (
  {template}
  ) array_row);
  '''
```

First query is written successfully

<img src="https://user-images.githubusercontent.com/66444859/226086946-ee1371b8-fa7b-4994-b644-75fbc92a2fc5.png" width=65%>

<img src="https://user-images.githubusercontent.com/66444859/226087205-82299b66-6584-4bba-bf7b-c3df021ffdd7.png" width=65%>

With proper query

<img src="https://user-images.githubusercontent.com/66444859/226087307-892a82a3-b7b4-4639-807e-650893a3f194.png" width=65%>

Frontend after adding query

<img src="https://user-images.githubusercontent.com/66444859/226087366-97f464b4-baea-4b08-8b26-62dc2f177ff0.png" width=50%>






