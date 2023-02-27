# Week 1 — App Containerization

### Required Homework

#### Containerize Application (Dockerfiles, Docker Compose)
 
I was able to containerize the frontend-react application following the Week 1 Live Streamed video and connect to it on browser: 

<img src="https://user-images.githubusercontent.com/66444859/221442458-6de5be75-6f35-4cab-a6e1-bd68eda86023.png" width="70%" >

Backend-flask

<img src="https://user-images.githubusercontent.com/66444859/221443427-1284fd16-2e80-448d-b5dc-aa88c6a921fa.png" width=70% >


Proof that ports are open and containers are running: 
<img src="https://user-images.githubusercontent.com/66444859/221442904-3d4d8297-1204-4195-9320-fcaef943eddf.png" width=80% >

Here is the link to [backend](https://github.com/nargiza777/aws-bootcamp-cruddur-2023/blob/main/backend-flask/Dockerfile) and [frontend](https://github.com/nargiza777/aws-bootcamp-cruddur-2023/blob/main/frontend-react-js/Dockerfile) Dockerfiles. 

#### Document the Notification Endpoint for the OpenAI Document

I was able to document the Notification Endpoint for the OpenAI Document

Added the endpoint in openapi.yaml:

<img src="https://user-images.githubusercontent.com/66444859/221444536-c9aa1a48-9435-42b9-a580-a2b161902f87.png" width=50% >

Joined Cruddur and confirmed email: 

<img src="https://user-images.githubusercontent.com/66444859/221443249-26894815-f89b-4682-8cc7-d437c8fdbf51.png" width=50% >

#### Create the notification feature for Backend and Frontend 

Define new endpoint in backend application for notifications in app.py and notifications_activities.py:

<img src="https://user-images.githubusercontent.com/66444859/221443032-83cd80f3-5db7-441a-98e7-245395981df5.png" width=50% >

<img src="https://user-images.githubusercontent.com/66444859/221443330-c47a045c-4898-433a-9de0-09befb3a1be1.png" width=50% >

Backend endpoint came back

<img src="https://user-images.githubusercontent.com/66444859/221445660-59c40310-6c81-452c-b022-043897a7c387.png" width=50% >



Implemented Frontend notifications feed page

<img src="https://user-images.githubusercontent.com/66444859/221442979-22294c85-bdab-4cf8-9c70-a229afa0326e.png" width=70% >

#### Run DynamoDB Local Container and ensure it works

Implemented DynamoDB and PostgresQL in docker-compose.yaml file. 
Verified DynamoDB is working. Example of using DynamoDB local we got from Andrew's [repo](https://github.com/100DaysOfCloud/challenge-dynamodb-local).

Created a table with this command:

```
aws dynamodb create-table \
    --endpoint-url http://localhost:8000 \
    --table-name Music \
    --attribute-definitions \
        AttributeName=Artist,AttributeType=S \
        AttributeName=SongTitle,AttributeType=S \
    --key-schema AttributeName=Artist,KeyType=HASH AttributeName=SongTitle,KeyType=RANGE \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
    --table-class STANDARD
![image](https://user-images.githubusercontent.com/66444859/221452494-ebdd87a4-6921-48cc-a8d0-9f42e2833937.png)

```
<img src="https://user-images.githubusercontent.com/66444859/221452320-c069ce5c-6dd1-4417-9710-32c35a0620a7.png" width=50% >


Created an item with: 
```
aws dynamodb put-item \
    --endpoint-url http://localhost:8000 \
    --table-name Music \
    --item \
        '{"Artist": {"S": "No One You Know"}, "SongTitle": {"S": "Call Me Today"}, "AlbumTitle": {"S": "Somewhat Famous"}}' \
    --return-consumed-capacity TOTAL  ![image](https://user-images.githubusercontent.com/66444859/221452553-c9d454d6-dc1b-439c-9046-c11fb44f0c11.png)

```
<img width="650" alt="image" src="https://user-images.githubusercontent.com/66444859/221452588-72c17806-a28a-48d5-83cc-83b0c5110971.png">

List tables
```
aws dynamodb list-tables --endpoint-url http://localhost:8000
```

<img src="https://user-images.githubusercontent.com/66444859/221452721-ad486b1a-63eb-42c6-88e0-462d5b70c2f3.png" width=50% >


Get records:

```
aws dynamodb scan --table-name Music --query "Items" --endpoint-url http://localhost:8000
```
<img width="650" alt="image" src="https://user-images.githubusercontent.com/66444859/221453139-97c9391a-4011-47fc-a766-db0c5d63df13.png">

#### Run Postgres Container and ensure it works

Install a  postgres client library to interact with a server which we put in our gitpod.yaml or you can run commands from terminal.

```
- name: postgres
    init: |
      curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
      echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
      sudo apt update
      sudo apt install -y postgresql-client-13 libpq-dev
```
In this code, first we are adding **gpg** and **debion** package to our source list.

After running commands, we tried running "psql" command and got an error.

<img width="650" alt="image" src="https://user-images.githubusercontent.com/66444859/221456131-72b84ecd-fc86-43df-af8a-e951cb08eaeb.png">

Tried to run Compose Up. Closed the Gitpod workspace and open again, run Compose Up and still got the same error:
<img src="https://user-images.githubusercontent.com/66444859/221456395-267ed352-6227-41b6-8b24-5959a00c6af9.png" width=50% >

Go to Database Explorer on Gitpod, click "+" to connect > go to PostgreSQL, Connection Name: database connection > port: 5432 > password: password > click Connect

<img src="https://user-images.githubusercontent.com/66444859/221460691-951873a9-6490-45dd-952f-9e773130986d.png" width=50% >


Connect was successful, but client is still not working. 
Turns out we need to use "-h" or "--host" to specify localhost, when postgres is running on docker container. Run this command: 
```
psql -U postgres --host localhost
```
We are in Postgres now: 

<img src="https://user-images.githubusercontent.com/66444859/221460616-fdfa02ea-f7bd-4c3d-9c2f-79f945e1171c.png" width=50% >

<img width="550" alt="image" src="https://user-images.githubusercontent.com/66444859/221460994-3d7b1f95-181a-426b-83b4-e3a5919346e3.png">

Verified PostgresQL is working by running following commands:
```
\d
\t
\l
```
<img width="550" alt="image" src="https://user-images.githubusercontent.com/66444859/221461187-7095a840-9491-4f51-b60d-cbd944729260.png">


#### Ashish's Week 1 - Container Security Considerations

Created a new account on Snyk. Forked [snyk-labs](https://github.com/snyk-labs/docker-goof) repo to test for vulnurabilities.

<img src="https://user-images.githubusercontent.com/66444859/221463608-c0c13013-6e83-4b73-b1d2-5fafe7d67a9a.png" width=55% >

Imported test repo to Snyk

<img src="https://user-images.githubusercontent.com/66444859/221464906-52af0b31-d83d-43a7-b7b2-589cf7c56a29.png" width=55% >

Scanned my repo and Snyk found 287 Critical, 991 High, 1.3k meduim vulnurabilities.

<img src="https://user-images.githubusercontent.com/66444859/221464220-bfa7515d-6944-41df-8c18-6f1a91ec2556.png" width=55% >

Recommendations for vulnerable Dockerfile to upgrade docker image from 10.40 version to 14.21.3.

<img src="https://user-images.githubusercontent.com/66444859/221464459-bc2ef891-1804-4892-8d9a-c44f374303df.png" width=55% >


Created a new secret in AWS Secret Manager

<img src="https://user-images.githubusercontent.com/66444859/221465468-8d6e95df-9b52-46e7-8e5d-3dda3bac312e.png" width=60% >

Explored and activated AWS Inspector

<img src="https://user-images.githubusercontent.com/66444859/221466075-e1c83ac0-b742-485f-8a6c-1fa25c601f5a.png" width=55% >


### Homework Challenges

#### Run the Dockerfile CMD as an external script

Created a Python script with Dockerfile CMD commands:
```
import subprocess
subprocess.run(["python3", "-m", "flask", "run", "--host=0.0.0.0", "--port=4567"])
```
Updated Dockerfile for backend app with an external script:
```
FROM python:3.10-slim-buster
WORKDIR /backend-flask
COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt
COPY . .
ENV FLASK_ENV=development
EXPOSE ${PORT}
COPY script.py /
RUN chmod +x /script.py
ENTRYPOINT ["python3", "script.py"]
```
Verified backend app container running and was able to connect from browser.

Placeholder for branch with new Dockerfile

#### Push and tag an image to Dockerhub

Logged in to Docker hub: 

<img src="https://user-images.githubusercontent.com/66444859/221475006-39f3927e-8f52-4469-83df-794c46a66d78.png" width=60% >

Tagged fronend image: 

<img src="https://user-images.githubusercontent.com/66444859/221475140-4b696d52-1394-4354-8909-c31879555b52.png" width=60% >

Pushed frontend-react image to Dockerhub:

<img src="https://user-images.githubusercontent.com/66444859/221469281-a93d80db-c8e6-4bca-a9e6-44f95b0390e0.png" width=55% >

Pushed backend-flask image: 

<img src="https://user-images.githubusercontent.com/66444859/221475290-db774f2f-5ef7-4dae-950f-56dd0862aa78.png" width=55% >


Pushed postgres image to Dockerhub:

<img src="https://user-images.githubusercontent.com/66444859/221469011-71f563d4-65c9-4281-a252-9aa079e03fb9.png" width=55% >

Verified dynamodb image was pushed to Dockerhub:

<img src="https://user-images.githubusercontent.com/66444859/221468619-9530a6fe-5ae9-4213-9ca5-eea43b23681f.png" width=55% >

List of images on Dockerhub:

<img src="https://user-images.githubusercontent.com/66444859/221472800-9add97a0-e1ea-4191-bdd8-65099ec81860.png" width=55% >

#### Reference:

[Docker Compose Documentation](https://docs.docker.com/compose/gettingstarted/)

