# Week 5 — DynamoDB and Serverless Caching

### DynamoDB Utility Scrips

We need to install Boto - AWS SDK for Python. Add to ```requirements.txt```.

```
boto3	
python-dateutil
```

```pip install -r requirements.txt```

We will be running DynamoDB locally. Run ```Compose Up``` to bring up containers. 

#### Create a Table in DynamoDB

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb/client/create_table.html

We will use SDK for that. In ```ddb/schema-load```

```
import boto3
import sys

attrs = {
   'endpoint_url': 'http://localhost:800'
}
if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}

ddb = boto3.client('dynamodb',**attrs)

table_name = "cruddur-messages"

response = ddb.create_table(
    TableName = table_name,
    AttributeDefinitions=[
        {
            'AttributeName': 'pk',
            'AttributeType': 'S'
        },
        {
            'AttributeName': 'sk',
            'AttributeType': 'S'
        }
    ],
    KeySchema=[
        {
            'AttributeName': 'pk',
            'KeyType': 'HASH'
        },
        {
            'AttributeName': 'sk',
            'KeyType': 'RANGE'
        }
    ],
    
    # GlobalSecondaryIndexes=[
    # ],
    BillingMode='PROVISIONED',
    ProvisionedThroughput={
        'ReadCapacityUnits': 5,
        'WriteCapacityUnits': 5
    }  
)
print(response)
```
Run ```./bin/ddb/schema-load```

Tabe was created

<img src="https://user-images.githubusercontent.com/66444859/227675540-440f848a-9722-4a05-a570-6c4019393dad.png" width=85%>

To list tables we will create a script ```/ddb/list-tables```

```
#! /usr/bin/bash
set -e # stop if it fails at any point

if [ "$1" = "prod" ]; then
  ENDPOINT_URL=""
else
  ENDPOINT_URL="--endpoint-url=http://localhost:8000"
fi

aws dynamodb list-tables $ENDPOINT_URL \
--query TableNames \
--output table
```

Run command to list tables ```./bin/ddb/list-tables```

<img src="https://user-images.githubusercontent.com/66444859/227678184-598a1eab-e157-45cb-9b6c-407cef9c6699.png" width=65%>

Create ```/ddb/setup```

```
#! /usr/bin/bash
set -e # stop if it fails at any point

CYAN='\033[1;36m'
NO_COLOR='\033[0m'
LABEL="db-setup"
printf "${CYAN}== ${LABEL}${NO_COLOR}\n"

bin_path="$(realpath .)/bin"

source "$bin_path/db/drop"
source "$bin_path/db/create"
source "$bin_path/db/schema-load"
source "$bin_path/db/seed"
```

To delete tables we will create a script ```/ddb/drop```

```
#! /usr/bin/bash

set -e # stop if it fails at any point

if [ -z "$1" ]; then
  echo "No TABLE_NAME argument supplied eg ./bin/ddb/drop cruddur-messages prod "
  exit 1
fi
TABLE_NAME=$1

if [ "$2" = "prod" ]; then
  ENDPOINT_URL=""
else
  ENDPOINT_URL="--endpoint-url=http://localhost:8000"
fi

echo "deleting table: $TABLE_NAME"

aws dynamodb delete-table $ENDPOINT_URL \
  --table-name $TABLE_NAME
```

Run ```./bin/ddb/drop cruddur-messages``` to delete table. It deleted table

<img src="https://user-images.githubusercontent.com/66444859/227679922-b6f0e491-fba4-4b6f-9f61-3e6b5dd18501.png" width=65%>


Create ```/ddb/seed```

```
#!/usr/bin/env python3

import boto3
import os
import sys
from datetime import datetime, timedelta, timezone
import uuid

current_path = os.path.dirname(os.path.abspath(__file__))
parent_path = os.path.abspath(os.path.join(current_path, '..', '..'))
sys.path.append(parent_path)
from lib.db import db

attrs = {
  'endpoint_url': 'http://localhost:8000'
}

# unset endpoint url for use with production database
if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}
ddb = boto3.client('dynamodb',**attrs)

def get_user_uuids():
  sql = """
  SELECT 
    users.uuid,
    users.display_name,
    users.handle
  FROM users
  WHERE
    users.handle IN(
      %(my_handle)s,
      %(other_handle)s
      )
  """
  users = db.query_array_json(sql,{
      'my_handle':  'andrewbrown',
      'other_handle': 'bayko'
    })
  my_user    = next((item for item in users if item["handle"] == 'andrewbrown'), None)
  other_user = next((item for item in users if item["handle"] == 'bayko'), None)
  results = {
    'my_user': my_user,
    'other_user': other_user
  }
  print('get_user_uuids')
  print(results)
  return results

def create_message_group(client,message_group_uuid, my_user_uuid, last_message_at=None, message=None, other_user_uuid=None, other_user_display_name=None, other_user_handle=None):
  table_name = 'cruddur-messages'
  record = {
    'pk':   {'S': f"GRP#{my_user_uuid}"},
    'sk':   {'S': last_message_at},
    'message_group_uuid': {'S': message_group_uuid},
    'message':  {'S': message},
    'user_uuid': {'S': other_user_uuid},
    'user_display_name': {'S': other_user_display_name},
    'user_handle': {'S': other_user_handle}
  }

  response = client.put_item(
    TableName=table_name,
    Item=record
  )
  print(response)

def create_message(client,message_group_uuid, created_at, message, my_user_uuid, my_user_display_name, my_user_handle):
  table_name = 'cruddur-messages'
  record = {
    'pk':   {'S': f"MSG#{message_group_uuid}"},
    'sk':   {'S': created_at },
    'message_uuid': { 'S': str(uuid.uuid4()) },
    'message': {'S': message},
    'user_uuid': {'S': my_user_uuid},
    'user_display_name': {'S': my_user_display_name},
    'user_handle': {'S': my_user_handle}
  }
  # insert the record into the table
  response = client.put_item(
    TableName=table_name,
    Item=record
  )
  # print the response
  print(response)

now = datetime.now(timezone.utc).astimezone()
message_group_uuid = "5ae290ed-55d1-47a0-bc6d-fe2bc2700399" #str(uuid.uuid4())
users = get_user_uuids()
 
create_message_group(
  client=ddb,
  message_group_uuid=message_group_uuid,
  my_user_uuid=users['my_user']['uuid'],
  other_user_uuid=users['other_user']['uuid'],
  other_user_handle=users['other_user']['handle'],
  other_user_display_name=users['other_user']['display_name'],
  last_message_at=now.isoformat(),
  message="this is a filler message"
)

create_message_group(
  client=ddb,
  message_group_uuid=message_group_uuid,
  my_user_uuid=users['other_user']['uuid'],
  other_user_uuid=users['my_user']['uuid'],
  other_user_handle=users['my_user']['handle'],
  other_user_display_name=users['my_user']['display_name'],
  last_message_at=now.isoformat(),
  message="this is a filler message"
)

conversation = """
Person 1: Have you ever watched Babylon 5? It's one of my favorite TV shows!
Person 2: Yes, I have! I love it too. What's your favorite season?
Person 1: I think my favorite season has to be season 3. So many great episodes, like "Severed Dreams" and "War Without End."
Person 2: Yeah, season 3 was amazing! I also loved season 4, especially with the Shadow War heating up and the introduction of the White Star.
...
Person 2: Definitely. I think his character is a great example of the show's ability to balance humor and heart, and to create memorable and beloved characters that fans will cherish for years to come.
"""

lines = conversation.lstrip('\n').rstrip('\n').split('\n') 
for i in range(len(lines)):
  if lines[i].startswith('Person 1: '):
    key = 'my_user'
    message = lines[i].replace('Person 1: ', '')
  elif lines[i].startswith('Person 2: '):
    key = 'other_user'
    message = lines[i].replace('Person 2: ', '')
  else:
    print(lines[i])
    raise 'invalid line'

  created_at = (now + timedelta(minutes=i)).isoformat()
  create_message(
    client=ddb,
    message_group_uuid= message_group_uuid,
    created_at=created_at,
    message=message,
    my_user_uuid=users[key]['uuid'],
    my_user_display_name=users[key]['display_name'],
    my_user_handle=users[key]['handle']
  )
```

We neeed to import lib and need to specify path fot that: 

```
current_path = os.path.dirname(os.path.abspath(__file__))
parent_path = os.path.abspath(os.path.join(current_path, '..', '..'))
sys.path.append(parent_path)
from lib.db import db
```

We will use SDK

```
attrs = {
  'endpoint_url': 'http://localhost:8000'
}

# unset endpoint url for use with production database
if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}
dynamodb = boto3.client('dynamodb',**attrs)
```

Create database

```.
./bin/db/create
./bin/db/schema-load
./bin/db/seed
```

Run ```./bin/ddb/seed```

<img src="https://user-images.githubusercontent.com/66444859/227682207-ebe914da-c2ff-4cf1-a84f-c64fda9dda3c.png" width=70%>


Returned data correctly.

Next step > Manipulate data to be our end structure. Add to ```/ddb/seed```

```
my_user    = next((item for item in users if item["handle"] == 'andrewbrown'), None)
other_user = next((item for item in users if item["handle"] == 'bayko'), None)
results = {
  'my_user': my_user,
  'other_user': other_user
}
print('get_user_uuids')
print(results)
```


Run ```./bin/ddb/seed```. 
It shows ```my-user``` as Andrew Brown and ```other_user``` as Andrew Bayko

<img src="https://user-images.githubusercontent.com/66444859/227682584-25d58267-c065-4881-beff-7737d84f3c6e.png" width=70%>

We need to create message groups. 

```
create_message_group(
  client=ddb,
  message_group_uuid=message_group_uuid,
  my_user_uuid=users['my_user']['uuid'],
  other_user_uuid=users['other_user']['uuid'],
  other_user_handle=users['other_user']['handle'],
  other_user_display_name=users['other_user']['display_name'],
  last_message_at=now.isoformat(),
  message="this is a filler message"
)

create_message_group(
  client=ddb,
  message_group_uuid=message_group_uuid,
  my_user_uuid=users['other_user']['uuid'],
  other_user_uuid=users['my_user']['uuid'],
  other_user_handle=users['my_user']['handle'],
  other_user_display_name=users['my_user']['display_name'],
  last_message_at=now.isoformat(),
  message="this is a filler message"
)
```


```
def create_message_group(client,message_group_uuid, my_user_uuid, last_message_at=None, message=None, other_user_uuid=None, other_user_display_name=None, other_user_handle=None):
  table_name = 'cruddur-messages'
  record = {
    'pk':   {'S': f"GRP#{my_user_uuid}"},
    'sk':   {'S': last_message_at},
    'message_group_uuid': {'S': message_group_uuid},
    'message':  {'S': message},
    'user_uuid': {'S': other_user_uuid},
    'user_display_name': {'S': other_user_display_name},
    'user_handle': {'S': other_user_handle}
  }

  response = client.put_item(
    TableName=table_name,
    Item=record
  )
  print(response)
```
```
def create_message(client,message_group_uuid, created_at, message, my_user_uuid, my_user_display_name, my_user_handle):
  # Entity # Message Group Id
  record = {
    'pk':   {'S': f"MSG#{message_group_uuid}"},
    'sk':   {'S': created_at },
    'message_uuid': { 'S': str(uuid.uuid4()) },
    'message': {'S': message},
    'user_uuid': {'S': my_user_uuid},
    'user_display_name': {'S': my_user_display_name},
    'user_handle': {'S': my_user_handle}
  }
  # insert the record into the table
  table_name = 'cruddur-messages'
  response = client.put_item(
    TableName=table_name,
    Item=record
  )
  # print the response
  print(response)
```

Run these commands

```
 ./bin/ddb/schema-load
 ./bin/ddb/list-tables
  ./bin/ddb/seed
```

It created a bunch of messages. 

To make sure data is there we can run scan. Create ```/ddb/scan``` script:

```
#!/usr/bin/env python3

import boto3

attrs = {
  'endpoint_url': 'http://localhost:8000'
}
ddb = boto3.resource('dynamodb',**attrs)
table_name = 'cruddur-messages'

table = ddb.Table(table_name)
response = table.scan()

print('========')
print(response)

items = response['Items']
for item in items:
    print(item)
```

Run ```/ddb/scan``` and we are getting our data back. 

<img src="https://user-images.githubusercontent.com/66444859/227689765-d2673bd5-c607-471c-be61-931e0f2ad505.png" width=65%>

#### Query data - List conversations, Get Conversation

Create new folder ```/ddb/patterns```

New file ```get-conversation```

```
#!/usr/bin/env python3

import boto3
import sys
import json
import datetime

attrs = {
  'endpoint_url': 'http://localhost:8000'
}

if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}

dynamodb = boto3.client('dynamodb',**attrs)
table_name = 'cruddur-messages'

message_group_uuid = "5ae290ed-55d1-47a0-bc6d-fe2bc2700399"

# define the query parameters
query_params = {
  'TableName': table_name,
  'KeyConditionExpression': 'pk = :pkey',
  'ScanIndexForward': False,
  'Limit': 20,
  'ExpressionAttributeValues': {
    ':pkey': {'S': f"MSG#{message_group_uuid}"}
  },
  'ReturnConsumedCapacity': 'TOTAL'
}

# query the table
response = dynamodb.query(**query_params)



# print the items returned by the query
print(json.dumps(response, sort_keys=True, indent=2))

# print the consumed capacity
print(json.dumps(response['ConsumedCapacity'], sort_keys=True, indent=2))

items = response['Items']
reversed_array = items[::-1]

for item in reversed_array:
  sender_handle = item['user_handle']['S']
  message       = item['message']['S']
  timestamp     = item['sk']['S']
  dt_object = datetime.datetime.strptime(timestamp, '%Y-%m-%dT%H:%M:%S.%f%z')
  formatted_datetime = dt_object.strftime('%Y-%m-%d %I:%M %p')
  print(f'{sender_handle: <16}{formatted_datetime: <22}{message[:40]}...')
```

```
chmod u+x ./bin/ddb/patterns/get-conversation
./bin/ddb/patterns/get-conversation 
```

This is our data structure, we can see our ```TableName``` and ```Consumed Capacity```

<img src="https://user-images.githubusercontent.com/66444859/227690489-59ae2f4c-01c0-4ac0-bb41-2431dca7568e.png" width=85%>


Here is our conversation

<img src="https://user-images.githubusercontent.com/66444859/227690341-bfa84d48-9f98-46f6-a70a-d8eaa3b98175.png" width=65%>


Adding ```start_date``` and ```end_date``` to  ```patterns/get-conversation```

```
#!/usr/bin/env python3

import boto3
import sys
import json
import datetime

attrs = {
  'endpoint_url': 'http://localhost:8000'
}

if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}

dynamodb = boto3.client('dynamodb',**attrs)
table_name = 'cruddur-messages'

message_group_uuid = "5ae290ed-55d1-47a0-bc6d-fe2bc2700399"

# define the query parameters
query_params = {
  'TableName': table_name,
  'ScanIndexForward': False,
  'Limit': 20,
  'ReturnConsumedCapacity': 'TOTAL',
  'KeyConditionExpression': 'pk = :pk AND begins_with(sk,:year)',
  #'KeyConditionExpression': 'pk = :pk AND sk BETWEEN :start_date AND :end_date',
  'ExpressionAttributeValues': {
    ':year': {'S': '2023'},
    #":start_date": { "S": "2023-03-01T00:00:00.000000+00:00" },
    #":end_date": { "S": "2023-03-19T23:59:59.999999+00:00" },
    ':pk': {'S': f"MSG#{message_group_uuid}"}
  }
}


# query the table
response = dynamodb.query(**query_params)

# print the items returned by the query
print(json.dumps(response, sort_keys=True, indent=2))

# print the consumed capacity
print(json.dumps(response['ConsumedCapacity'], sort_keys=True, indent=2))

items = response['Items']
reversed_array = items[::-1]

for item in reversed_array:
  sender_handle = item['user_handle']['S']
  message       = item['message']['S']
  timestamp     = item['sk']['S']
  dt_object = datetime.datetime.strptime(timestamp, '%Y-%m-%dT%H:%M:%S.%f%z')
  formatted_datetime = dt_object.strftime('%Y-%m-%d %I:%M %p')
  print(f'{sender_handle: <12}{formatted_datetime: <22}{message[:40]}...')
```


Create ```patterns/list-conversation```

```
#!/usr/bin/env python3

import boto3
import sys
import json
import os

current_path = os.path.dirname(os.path.abspath(__file__))
parent_path = os.path.abspath(os.path.join(current_path, '..', '..', '..'))
sys.path.append(parent_path)
from lib.db import db

attrs = {
  'endpoint_url': 'http://localhost:8000'
}

if len(sys.argv) == 2:
  if "prod" in sys.argv[1]:
    attrs = {}

dynamodb = boto3.client('dynamodb',**attrs)
table_name = 'cruddur-messages'

def get_my_user_uuid():
  sql = """
    SELECT 
      users.uuid
    FROM users
    WHERE
      users.handle =%(handle)s
  """
  uuid = db.query_value(sql,{
    'handle':  'andrewbrown'
  })
  return uuid

my_user_uuid = get_my_user_uuid()
print(f"my-uuid: {my_user_uuid}")

# define the query parameters
query_params = {
  'TableName': table_name,
  'KeyConditionExpression': 'pk = :pk',
  'ExpressionAttributeValues': {
    ':pk': {'S': f"GRP#{my_user_uuid}"}
  },
  'ReturnConsumedCapacity': 'TOTAL'
}

# query the table
response = dynamodb.query(**query_params)

# print the items returned by the query
print(json.dumps(response, sort_keys=True, indent=2))
```

Get ```uuid``` from Postgres. 

```
./bin/db/connec
SELECT uuid, handle from users;
```

Every time we seed database we need to update this value.

```
my_user_uuid = " b02d00ce-df83-4373-9710-9868564a0809"
```

Or we can implement function to update this value. Put in ```list-conversations```:

```
def get_my_user_uuid():
  sql = """
  SELECT 
    users.uuid
  FROM users
  WHERE
    users.handle =%(handle)s,
  """
  uuid = db.query_value(sql,{
    'handle':  'andrewbrown',
    })
return uuid

my_user_uuid = get_my_user_uuid()
print(f"my-uuid:{my_user_uuid}")
```

Add to ```/lib/db.py```

```
def query_value(self,sql,params={}):
  self.print_sql('value',sql,params)
  with self.pool.connection() as conn:
    with conn.cursor() as cur:
      cur.execute(sql,params)
      json = cur.fetchone()
      return json[0]
```

Run ```./bin/ddb/patterns/list-conversations ```

Now we can see what's being passed in our queries.

<img src="https://user-images.githubusercontent.com/66444859/227696572-36759836-f91c-4f2d-be6e-561258c67824.png" width=65%>

<img src="https://user-images.githubusercontent.com/66444859/227696510-8bde55f6-b3a4-4d34-b4c9-f2812e7d926a.png" width=65%>





