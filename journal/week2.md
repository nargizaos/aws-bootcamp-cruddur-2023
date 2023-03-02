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

<img src="https://user-images.githubusercontent.com/66444859/222337755-2b11ed39-d09e-4190-bc6e-ccf8a25235fa.png" width=40% >

X-Ray traces group was created, which will group traces together with ```service("backend-flask")``` filter:

<img src="https://user-images.githubusercontent.com/66444859/222338469-dc8f2fda-39e5-40cc-a185-166ae72d178b.png" width=55% >

Create sampling rule
```aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json```

<img src="https://user-images.githubusercontent.com/66444859/222340536-9812a71d-91e0-410c-ba05-d407b4fa8a46.png" width=55% >

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



