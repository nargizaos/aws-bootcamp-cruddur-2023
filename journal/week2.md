# Week 2 — Distributed Tracing
### Required Homework

#### Instrument X-Ray

Added aws-sdk to ```app.py```:

<img src="https://user-images.githubusercontent.com/66444859/222324500-8aa1ce12-2732-4924-a4cd-74be188af47c.png" width=40% >

Install python dependencies:
```
cd backend-flask
pip install -r requirements.txt
```

Added to ```app.py```: 
```
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

xray_url = os.getenv("AWS_XRAY_URL")
xray_recorder.configure(service='Cruddur', dynamic_naming=xray_url)
XRayMiddleware(app, xray_recorder)
```
<img src="https://user-images.githubusercontent.com/66444859/222326635-0ff4c518-5fb5-4293-99f4-fa87b75a343f.png" width=60% >

##### Setup AWS X-Ray Resources
Add ```aws/json/xray.json```
```
{
  "SamplingRule": {
      "RuleName": "Cruddur",
      "ResourceARN": "*",
      "Priority": 9000,
      "FixedRate": 0.1,
      "ReservoirSize": 5,
      "ServiceName": "Cruddur",
      "ServiceType": "*",
      "Host": "*",
      "HTTPMethod": "*",
      "URLPath": "*",
      "Version": 1
  }
}
```
<img src="https://user-images.githubusercontent.com/66444859/222327256-cf997a2d-0566-49d4-be3d-b26372b7cdea.png" width=40% >

```
aws xray create-group \
   --group-name "Cruddur" \
   --filter-expression "service(\"backend-flask\")"
```

<img src="https://user-images.githubusercontent.com/66444859/222337755-2b11ed39-d09e-4190-bc6e-ccf8a25235fa.png" width=65% >

X-Ray traces group was created, which will group traces together with ```service("backend-flask")``` filter:

<img src="https://user-images.githubusercontent.com/66444859/222338469-dc8f2fda-39e5-40cc-a185-166ae72d178b.png" width=55% >

Create sampling rule
```aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json```

<img src="https://user-images.githubusercontent.com/66444859/222342893-bd1177a8-4e25-4ccc-875d-27699d47156c.png" width=55% >

Sampling rile was created

<img src="https://user-images.githubusercontent.com/66444859/222341367-08e04544-af52-44b1-a1fa-b8ed0d438343.png" width=55% >

##### Install X-Ray Daemon

Add Deamon Service to Docker Compose
```
  xray-daemon:
    image: "amazon/aws-xray-daemon"
    environment:
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
      AWS_REGION: "us-east-1"
    command:
      - "xray -o -b xray-daemon:2000"
    ports:
      - 2000:2000/udp
```
<img src="https://user-images.githubusercontent.com/66444859/222346226-344a38b5-3c1f-44a8-8928-5dc9d8eeacc1.png" width=50% >

Add these two env vars to our backend-flask in our ```docker-compose.yml``` file. Here are providing AWS X-Ray url and daemon address
```
      AWS_XRAY_URL: "*4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}*"
      AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"
```
Run ```docker compose up```

Backend and xray-daemon containers are not running

<img src="https://user-images.githubusercontent.com/66444859/222346892-62bfe716-5fd8-4f63-8210-486cdbe74819.png" width=50% >

Checked backend container logs and it shows that ```\"app" is not defined```

<img src="https://user-images.githubusercontent.com/66444859/222347483-b60ff5ae-d34a-4240-ad14-c22741626a9a.png" width=50% >

We moved ```XRayMiddleware(app, xray_recorder)``` under "app" in ```app.py```

<img src="https://user-images.githubusercontent.com/66444859/222347911-9a4b750f-fac6-41fd-936a-0c2b1528afae.png" width=45% >

Re-run ```compose up``` and backend and xray-daemon containers are running. 
Opened backend on browser and was able to connect. Hit endpoint multiple times. 

Looking in backend-flask logs, Andrew got xray errors saying: ```GetSamplingRules operation: Bad Gateway```.
But in my logs I did not get any errors.
In xray-daemon logs Andrew got error: ```send request failed: ... no such host```

This is what I got in my xray-daemon logs: 

<img src="https://user-images.githubusercontent.com/66444859/222350836-200c0aad-50e7-464f-8fcc-a1d2e4d5519b.png" width=65% >

Looks like Andrew misspelled AWS region name in ```docker-compose.yaml```.

In order to find out it's being delivered into X-Ray, open xray-daemon logs it is showing that batch of segments were successfully sent.

<img src="https://user-images.githubusercontent.com/66444859/222352136-43aa5f10-6175-4451-90a4-c72d24f26e7e.png" width=65% >

Go to AWS console > X-Ray > Traces - we can see some data

<img src="https://user-images.githubusercontent.com/66444859/222353321-878e83a8-befd-48ce-89e6-16ef43b8b76b.png" width=65% >

<img src="https://user-images.githubusercontent.com/66444859/222352883-a43360b5-97e2-4943-ac72-e57876e4cc44.png" width=49% >

If we click on one of the traces, we can see Trace Map

<img src="https://user-images.githubusercontent.com/66444859/222353593-2d930bb7-6195-4e71-b780-dcc68db3fcd0.png" width=49% >

Here is our span:

<img src="https://user-images.githubusercontent.com/66444859/222353968-76725b6f-4d55-43e4-a94e-0b0135482f64.png" width=65% >

##### Start a custom segment/subsegment
From [AWS X-Ray repo](https://github.com/aws/aws-xray-sdk-python)

Added custom segment to ```user_activities.py```

<img src="https://user-images.githubusercontent.com/66444859/222876371-6874b335-8dd3-4728-9638-2efbd2041f6e.png" width=49% >

<img src="https://user-images.githubusercontent.com/66444859/222876395-9861b956-ff61-4405-bb72-4da159c6b8e9.png" width=49% >

Ran the query from X-Ray Traces and got error:

<img src="https://user-images.githubusercontent.com/66444859/222876450-eeb5c87d-d862-4733-b4d4-ee358d8b1d9c.png" width=55% >

Re-ran ```Compose Up```, hit endpoint multiple times, checked from AWS X-ray Traces, still getting 4xx errors, will look into it later.






##### Referrence 

https://github.com/aws/aws-xray-sdk-python
