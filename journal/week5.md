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

Table was created

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

```
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
./bin/db/connect
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

Add to ```/lib/db.py```:

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

### Implement (Pattern A) Listing Messages in Message Group into Application

Create new file in /lib/ ```/lib/ddb.py```

```
import boto3
import sys
from datetime import datetime, timedelta, timezone
import uuid
import os

class Ddb:
  def client():
    endpoint_url = os.getenv("AWS_ENDPOINT_URL")
    if endpoint_url:
      attrs = { 'endpoint_url': endpoint_url }
    else:
      attrs = {}
    dynamodb = boto3.client('dynamodb',**attrs)
    return dynamodb

  def list_message_groups(client,my_user_uuid):
    table_name = 'cruddur-messages'
    query_params = {
      'TableName': table_name,
      'KeyConditionExpression': 'pk = :pk',
      'ScanIndexForward': False,
      'Limit': 20,
      'ExpressionAttributeValues': {
        ':pk': {'S': f"GRP#{my_user_uuid}"}
      }
    }
    print('query-params')
    print(query_params)
    print('client')
    print(client)

    # query the table
    response = client.query(**query_params)
    items = response['Items']
    
    results = []
    for item in items:
      last_sent_at = item['sk']['S']
      results.append({
        'uuid': item['message_group_uuid']['S'],
        'display_name': item['user_display_name']['S'],
        'handle': item['user_handle']['S'],
        'message': item['message']['S'],
        'created_at': last_sent_at
      })
    return results
```

Create new ```cognito``` folder inn ```bin``` directory. We will implement Cognito so that dynamodb will pick up our User ID from Cognito. 

Command to list users from AWS CLI:

```
aws cognito-idp list-users --user-pool-id=us-east-1_sKlh00oHV
```

Export Cognito Envs:

```
export AWS_COGNITO_USER_POOL_ID=us-east-1_sKlh00oHV
gp env AWS_COGNITO_USER_POOL_ID=us-east-1_sKlh00oHV
```

New file ```list-users```:

```
#!/usr/bin/env python3

import boto3
import os
import json

userpool_id = os.getenv("AWS_COGNITO_USER_POOL_ID")
client = boto3.client('cognito-idp')
params = {
  'UserPoolId': userpool_id,
  'AttributesToGet': [
      'preferred_username',
      'sub'
  ]
}
response = client.list_users(**params)
users = response['Users']

print(json.dumps(users, sort_keys=True, indent=2, default=str))

dict_users = {}
for user in users:
  attrs = user['Attributes']
  sub    = next((a for a in attrs if a["Name"] == 'sub'), None)
  handle = next((a for a in attrs if a["Name"] == 'preferred_username'), None)
  dict_users[handle['Value']] = sub['Value']

print(json.dumps(dict_users, sort_keys=True, indent=2, default=str))
```

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cognito-idp/client/list_users.html

Run the script with ```./bin/cognito/list-users ```

<img src="https://user-images.githubusercontent.com/66444859/228637019-4dd0f2ba-ed1b-4886-9c49-1762caae2350.png" width=65%>

Next step is to create a new script to update User names in our database ```backend-flask/bin/db/update_cognito_user_ids```

```
#!/usr/bin/env python3

import boto3
import os
import sys

print("== db-update-cognito-user-ids")

current_path = os.path.dirname(os.path.abspath(__file__))
parent_path = os.path.abspath(os.path.join(current_path, '..', '..'))
sys.path.append(parent_path)
from lib.db import db

def update_users_with_cognito_user_id(handle,sub):
  sql = """
    UPDATE public.users
    SET cognito_user_id = %(sub)s
    WHERE
      users.handle = %(handle)s;
  """
  db.query_commit(sql,{
    'handle' : handle,
    'sub' : sub
  })

def get_cognito_user_ids():
  userpool_id = os.getenv("AWS_COGNITO_USER_POOL_ID")
  client = boto3.client('cognito-idp')
  params = {
    'UserPoolId': userpool_id,
    'AttributesToGet': [
        'preferred_username',
        'sub'
    ]
  }
  response = client.list_users(**params)
  users = response['Users']
  dict_users = {}
  for user in users:
    attrs = user['Attributes']
    sub    = next((a for a in attrs if a["Name"] == 'sub'), None)
    handle = next((a for a in attrs if a["Name"] == 'preferred_username'), None)
    dict_users[handle['Value']] = sub['Value']
  return dict_users


users = get_cognito_user_ids()

for handle, sub in users.items():
  print('----',handle,sub)
  update_users_with_cognito_user_id(
    handle=handle,
    sub=sub
  )
```

Update ```setup``` script with:

```
source "$bin_path/db/update_cognito_user_ids"
```

We will bring up our database:

```./bin/db/setup```

We ran into ```import-im6.q16: unable to open X server `' @ error/import.c/ImportImageCommand/359.``` error. 

<img src="https://user-images.githubusercontent.com/66444859/228638876-14fc775e-2d6c-4804-b7b4-783bd4acd661.png" width=65%>

Looks like we forgot to make new file executable: ```chmod u+x ./bin/db/update_cognito_user_ids```.

We are still having previuos error. We will try to execute ```./bin/db/update_cognito_user_ids``` separately. 

<img src="https://user-images.githubusercontent.com/66444859/228639773-a37d7294-a332-4ea3-abe8-ed7ac7ebd100.png" width=65%>


Update this line in ```/lib/db.py``` and run ```./bin/db/update_cognito_user_ids``` again.

```
self.print_sql('commit with returning',sql,params)
```

Now it's showing what exactly it's passing. 

<img src="https://user-images.githubusercontent.com/66444859/228640709-855b6255-9d83-4b20-946c-628b0ddebe26.png" width=65%>

But ```./bin/db/setup``` command is still not working. In ```/bin/db/setup``` change 

```source "$bin_path/db/update_cognito_user_ids"``` to ```python "$bin_path/db/update_cognito_user_ids"```. 

The error is caused by trying to run a Python script as a Bash script.

Update ```message_groups.py```

```
from datetime import datetime, timedelta, timezone

from lib.ddb import Ddb
from lib.db import db

class MessageGroups:
  def run(cognito_user_id):
    model = {
      'errors': None,
      'data': None
    }

    sql = db.template('users','uuid_from_cognito_user_id')
    my_user_uuid = db.query_value(sql,{
      'cognito_user_id': cognito_user_id
    })

    print(f"UUID: {my_user_uuid}")

    ddb = Ddb.client()
    data = Ddb.list_message_groups(ddb, my_user_uuid)
    print("list_message_groups:",data)

    model['data'] = data
    return model
```

Update ```HomeFeedPage.js``` to update token

```
headers: {
  Authorization: `Bearer ${localStorage.getItem("access_token")}`
},
```

Create ```CheckAuth.js``` in ```frontend/src/lib```:

```
import { Auth } from 'aws-amplify';

const checkAuth = async (setUser) => {
  Auth.currentAuthenticatedUser({
    // Optional, By default is false. 
    // If set to true, this call will send a 
    // request to Cognito to get the latest user data
    bypassCache: false 
  })
  .then((user) => {
    console.log('user',user);
    return Auth.currentAuthenticatedUser()
  }).then((cognito_user) => {
      setUser({
        display_name: cognito_user.attributes.name,
        handle: cognito_user.attributes.preferred_username
      })
  })
  .catch((err) => console.log(err));
};

export default checkAuth;
```

```MessageGroups``` are not returning any data.

<img src="https://user-images.githubusercontent.com/66444859/228963370-13733f45-ba07-4012-a718-b94e258148e2.png" width=65%>


```
./bin/ddb/schema-load 
./bin/ddb/seed 
```

For me it's still showing error when I do inspect. But when I run ```./bin/ddb/patterns/list-conversations ``` it's showing list of conversations. We might have missed something in our queries. Let's look into our ```ddb.py```. 

Looks like we didn't update ``` AWS_ENDPOINT_URL``` and we will update it now in ```docker-compose.yaml```:

```
 AWS_ENDPOINT_URL: "http://dynamodb-local:8000"
 ```
 
 ### Implement (Pattern B) Listing Messages Group into Application
 
 Updates in ```backend-flask/db/seed.sql```
 
 Line 4: deleted ```Andrew Brown value and added my User Value/handle```
 
 Line 12: 
```(SELECT uuid from public.users WHERE users.handle = 'andrewbrown' LIMIT 1),``` >>> ```(SELECT uuid from public.users WHERE users.handle = 'nargizaosmon' LIMIT 1),```


Updates in ```backend-flask/ddb/seed```

```'my_handle': 'andrewbrown',``` >> ```'my_handle': 'nargizaosmon',```

```my_user = next((item for item in users if item["handle"] == 'andrewbrown'), None)``` >>>
```user = next((item for item in users if item["handle"] == 'nargizaosmon'), None)```

Updates in ```backend-flask/ddb/patterns/list-conversations```

'handle': 'andrewbrown' >>> 'handle': 'nargizaosmon'

Here is the Message Group:

<img width="850" alt="image" src="https://user-images.githubusercontent.com/66444859/236047144-f78d8e79-6bf3-4734-8c91-e809c0f16644.png">

<img width="850" alt="image" src="https://user-images.githubusercontent.com/66444859/236047419-44aab8dc-41ed-4324-8030-fedb4fa47258.png">


List of users in DB: 

<img width="850" alt="image" src="https://user-images.githubusercontent.com/66444859/236047742-74c5e663-0881-4b6b-9295-bfce221edada.png">

List of conversations: 

<img width="650" alt="image" src="https://user-images.githubusercontent.com/66444859/236047812-7fab4c4d-4ac3-4e4b-aa9e-ec7d5ad7003d.png">

<img width="650" alt="image" src="https://user-images.githubusercontent.com/66444859/236047952-2a02e973-b497-45dd-b854-0d32fbea9dad.png">


### Create Message in existing Message Group

In ```/backend-flask/bin/ddb/seed``` change time format from 

```(now + timedelta(minutes=i)).isoformat()``` 
to  ```(now + timedelta(hours=-3) + timedelta(minutes=i)).isoformat()```

After changing you will have to drop and recreate the ddb table: 
```
1) ./bin/ddb/drop cruddur-messages 
2) ./bin/ddb/schema-load
3)./bin/ddb/seed 
```
and make sure you do not get any errors when running this binaries.

Create ```/backend-flask/db/sql/users/create_message_users.sql```

```
SELECT 
  users.uuid,
  users.display_name,
  users.handle,
  CASE users.cognito_user_id = %(cognito_user_id)s
  WHEN TRUE THEN
    'sender'
  WHEN FALSE THEN
    'recv'
  ELSE
    'other'
  END as kind
FROM public.users
WHERE
  users.cognito_user_id = %(cognito_user_id)s
  OR 
  users.handle = %(user_receiver_handle)s
```

Open Message Group and try sending a message. Message was sent successfully.

<img width="850" alt="image" src="https://user-images.githubusercontent.com/66444859/236049377-2ff4bf03-121f-4e52-97cc-f006d34ddcaa.png">


### Implement Conversations with DynamoDB - Create New Conversation - Pattern C 

Add new path in App.js in frontend-react

```
{
  path: "/messages/new/:handle",
  element: <MessageGroupNewPage />
},
```

And add new import:

```import MessageGroupNewPage from './pages/MessageGroupNewPage';```


In ```frontend-react-js/pages``` create new file for new Message Group ```MessageGroupNewPage.js``` [see content here](https://github.com/nargiza777/aws-bootcamp-cruddur-2023/blob/main/frontend-react-js/src/pages/MessageGroupNewPage.js)


Add new User in our seed data ```backend-flask/db/seed.sql```

```('Londo Mollari', 'lmollari@centari.com', 'londo' ,'MOCK');```

Connect to database and insert Londo user:

```
./bin/db/connect
INSERT INTO public.users (display_name, email, handle, cognito_user_id) VALUES ('Londo Mollari', 'lmollari@centari.com', 'londo' ,'MOCK');
```

<img width="950" alt="image" src="https://user-images.githubusercontent.com/66444859/236051747-2d116c34-fa18-49b9-8b77-0035eb45d399.png">

We need to create API Endpoint for params. Add this to ```backend-flask/app.py```

```
from services.users_short import *

@app.route("/api/users/@<string:handle>/short", methods=['GET'])
def data_users_short(handle):
data = UsersShort.run(handle)
return data, 200
```

Create new file backend-flask/services/user_short.py ([code](https://github.com/nargiza777/aws-bootcamp-cruddur-2023/blob/main/backend-flask/services/users_short.py))

Create new file ```backend-flask/db/sql/users/short.sql```

```
SELECT
  users.uuid,
  users.handle,
  users.display_name
FROM public.users
WHERE 
  users.handle = %(handle)s
```

Create new file for New Item in Message Groups ```frontend-react-js/src/components/MessageGroupNewItem.js``` ([code](https://github.com/nargiza777/aws-bootcamp-cruddur-2023/blob/main/frontend-react-js/src/components/MessageGroupNewItem.js))





