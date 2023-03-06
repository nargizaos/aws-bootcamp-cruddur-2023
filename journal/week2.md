# Week 2 — Distributed Tracing
## Required Homework

### Distributed Tracing with HoneyComb

Open HoneyComb > Create new environment called ```bbotcamp```. 
Get ```API key``` from HB and export: 

```
export HONEYCOMB_API_KEY=""
export HONEYCOMB_SERVICE_NAME="Cruddur"
gp env HONEYCOMB_API_KEY=""
gp env HONEYCOMB_SERVICE_NAME="Cruddur"
```

Add OTEL Env vars to ```docker-compose.yaml```
```
OTEL_SERVICE_NAME: 'backend-flask'
OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"
OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=${HONEYCOMB_API_KEY}"
```

Add the following files to our ```requirements.txt```
```
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests
```
Install these dependencies:
```pip install -r requirements.txt```

Add to the ```app.py```
```
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
```

```
# Initialize tracing and an exporter that can send data to Honeycomb
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)
```

Put in ```app.py``` below ```app = Flask(__name__)```:
```
# Initialize automatic instrumentation with Flask
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```

Add SpanProcessor to ```app.py```:
```from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter```

```
#Show this in the logs within backend-flask
simple_processor = SimpleSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(simple_processor)
```

Run ```Compose Up```

You can check which Environment is using your API key on: ```honeycomb-whoami.glitchme```

<img src="https://user-images.githubusercontent.com/66444859/223024244-20248ea9-f033-4332-a98f-0301030a2ee5.png" width=40% >

We connected to backend-flask shell and checked Env Vars ```env | grep HONEY``` and it's not there: 

<img src="https://user-images.githubusercontent.com/66444859/223024502-a1cd53ea-577e-49cc-9e45-c79efc9a4fbc.png" width=40% >

In order to solve it, we can try comitting code, closing and opening Gitpod workspace again.

If we open HCM home we see there were some traces created

<img src="https://user-images.githubusercontent.com/66444859/223025073-0298cf9f-8667-4c28-aefb-9976c9505443.png" width=55% >

<img src="https://user-images.githubusercontent.com/66444859/223025161-4468ffa5-7ab0-41c2-8f48-b70a7ced298e.png" width=20% >


Hard coding a span
Add this to ```app.py```
```
from opentelemetry import trace
tracer = trace.get_tracer("home.activities")
```

And add this to ```home_activities.py```:
```
def run(logger):
  with tracer.start_as_current_span("home-activites-mock-data"):
    span = trace.get_current_span()
```

Open backend home page and make sure it's working.

On HCM home page see if it's getting data and traces. New ```home-activities-mock-data``` trace was created

<img src="https://user-images.githubusercontent.com/66444859/223026895-febc4800-772e-4f58-9258-6e4f2e45c575.png" width=55% >

<img src="https://user-images.githubusercontent.com/66444859/223027139-5a74b455-f107-469c-ac28-f5cc9a66b494.png" width=60% >

Add attribute to the span, add to ```home_activities.py```:
```
 now = datetime.now(timezone.utc).astimezone()
 span.set_attribute("app.now", now.isoformat())
```

```
 span.set_attribute("app.result_length", len(results))
```
Create custom query

<img src="https://user-images.githubusercontent.com/66444859/223030709-19608c15-4b93-44c7-bedd-61d6cb37e494.png" width=55% >

<img src="https://user-images.githubusercontent.com/66444859/223030847-3e6fc0e6-7b3e-4488-ac31-435e1711c74a.png" width=55% >

See traces for last 10 minutes

<img src="https://user-images.githubusercontent.com/66444859/223031013-45bba875-5967-4e23-b028-c8faf0108bbc.png" width=55% >

Try one more query for ```app.result_length exists```

<img src="https://user-images.githubusercontent.com/66444859/223031292-ae36047f-54cf-41bc-8500-c6885ad6cc89.png" width=55% >

Look into latency with ```HEATMAP(duration_ms)```

<img src="https://user-images.githubusercontent.com/66444859/223031447-09531f95-20cb-43c1-b148-ddb63239762e.png" width=55% >

<img src="https://user-images.githubusercontent.com/66444859/223031485-70e2b901-afd5-4d09-8f26-d291334f84bc.png" width=55% >


### Instrument X-Ray

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

Sampling rule was created

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

#### Fixing X-Ray
Watched Andrew's video for X-Ray Subsegments Solved.

In irder to continue fixing errors, we need to bring back commented out ```segments``` in ```app.py``` and ```user_activities.py```.
Since we want Traces to be created for user ```@andrewbronw```, we want to hit ```backendurl\@andrewbron``` endpoint.

Hit ```backendurl\@andrewbron``` endpoint multiple times, Go to AWS X-Ray > Traces > see if new traces were created for this endpoint. 

We checked the logs for ```-xray-daemon``` container for ```home``` page and it's sending segments, but it's not sending segmnents for ```\userpage```.

We tried using ChatGPT to generate code using AWS X-Ray SDK: implement a flask application endpoint to use AWS SDK X-Ray.
It produced similar code as we had, additionally it produced ```capture``` method to use as a subsegment that will be associated with our endpoint function. 
Added this line to ```/api/activities/home``` route in ```app.py```:
```@xray_recorder.capture('activities_home')```

as well as to ```UserActivities```:
```@xray_recorder.capture('activities_show')```

Update ```home``` and ```\userfeed``` page multiple times and see if we got some data. Go to X-Ray Traces > 


<img src="https://user-images.githubusercontent.com/66444859/223016233-b3c64ee7-216e-4b72-a75a-ee1b632dd41b.png" width=40% >

Placeholder for fix, will come back later and add steps.


### Configure custom logger to send to CloudWatch Logs

Added to the ```requirements.txt```
```watchtower```
```pip install -r requirements.txt```

Watchtower is a log handler for AWS CloudWatch Logs.

In ```app.py``` added:
```
import watchtower
import logging
from time import strftime
```
It will set up log group in CloudWatch Logs called **cruddur**
```
# Configuring Logger to Use CloudWatch
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
LOGGER.info("some message")
```
<img src="https://user-images.githubusercontent.com/66444859/222877769-68f21098-7d2e-432c-8794-931fc2c1d11e.png" width=55% >


```
@app.after_request
def after_request(response):
    timestamp = strftime('[%Y-%b-%d %H:%M]')
    LOGGER.error('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
    return response
``` 
Set the env var in your backend-flask for ```docker-compose.yml```
```
      AWS_DEFAULT_REGION: "${AWS_DEFAULT_REGION}"
      AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```

Added ```logger``` value in ```/api/activities/home``` in ```app.py```: 

<img src="https://user-images.githubusercontent.com/66444859/222877758-e45018ae-ac70-4753-927a-c902d94bee5c.png" width=55% >

Backend/api/activities/home came up working, hit the endpoint multiple times.

Go to CloudWatch from AWS console > Log groups > we will see cruddur log group

<img src="https://user-images.githubusercontent.com/66444859/222877965-e8a93743-cf95-468c-ae24-86bc33547f6f.png" width=55% >

<img src="https://user-images.githubusercontent.com/66444859/222878002-bc7b8595-8862-4d32-a5db-ec2e290943a9.png" width=55% >

It's showing ```HomeActivities```

<img src="https://user-images.githubusercontent.com/66444859/222878212-f8adbeab-2437-43e0-b5f2-d0a06dfee754.png" width=55%>


### Rollbar

Added to ```requirements.txt```:
```
blinker
rollbar
```
```pip install -r requirements.txt```

 Set our Rollbar access token: 
 ```
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```

Added to backend-flask for ```docker-compose.yml```
```ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"```

Imported for Rollbar
```
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```

Added import for Rollbar to ```app.py```:
```
import os
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception
```
```
rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')
@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)
```

Added endpoint for tesing Rollbar to ```app.py```:
```
@app.route('/rollbar/test')
def rollbar_test():
    rollbar.report_message('Hello World!', 'warning')
    return "Hello World!"
```

Ran ```Compose Up``` to bring up our apps.
When browsing backend ran into error ```'app' is not defined```, container did not come up healthy and wouldn't open from browser. Turns out we placed 
our code for Rollbar initialization above ```app```. Moved code below ```app``` in ``app.py```. And Compose Up. 

Home page came up clean

<img src="https://user-images.githubusercontent.com/66444859/222991340-3a343ff5-b704-43c3-887a-a0c794624fcf.png" width=45%>

Opened ```/rollbar/test``` endpoint

<img src="https://user-images.githubusercontent.com/66444859/222991391-4b0d39e1-e2bf-4519-ade4-ef7736c15102.png" width=65%>

Rollbar page is showing that it is listening:

<img src="https://user-images.githubusercontent.com/66444859/222991484-c13ace2b-0f7f-4e1c-87e8-1521c69c81ee.png" width=55%>

Checked the logs for backend-flask

<img src="https://user-images.githubusercontent.com/66444859/222991621-d7fe719d-af2e-4736-97b8-30aa4e102c0a.png" width=55%>

Attached to backend-flask shell and checked ```env``` variables to be set and it's showing it's not set. 
Closed workspaces and attached to workspace again. 
Commented out X-Ray processor for now: 
```
simple_processor = SimpleSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(simple_processor)
```

Added Rollbar Access Token to ```docker-compose.yaml```:
```ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"```

Logs are showing Rollbar test: 

<img src="https://user-images.githubusercontent.com/66444859/222994041-5eaa9d50-81ba-4793-b13a-ac6da9804a6c.png" width=55%>

Now we can see new Item in Rollbar

<img src="https://user-images.githubusercontent.com/66444859/222994180-aa22fd53-d5a9-44af-839a-00d98929b43c.png" width=55%>

If we click ```Hello World!``` item

<img src="https://user-images.githubusercontent.com/66444859/222994230-38605a68-6423-49ef-a598-55f67e6b34fe.png" width=55%>

<img src="https://user-images.githubusercontent.com/66444859/222994248-55b3b1ec-40dd-4491-861c-f9fa19560fdf.png" width=55%>

Let's change the code and make an error occur. Commented out ```return results``` from ```home_activities.py```.

Go to Rollbar > Items and will see an error: 

<img src="https://user-images.githubusercontent.com/66444859/222994498-db46d733-58b4-4c07-baa2-5f7f2d3e8f3b.png" width=55%>

<img src="https://user-images.githubusercontent.com/66444859/222995348-7bbdea7a-1268-4e71-981d-7cc202a2bba8.png" width=55%>







##### Referrence 

https://github.com/aws/aws-xray-sdk-python
