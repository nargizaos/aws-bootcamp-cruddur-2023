# Week 3 — Decentralized Authentication

### Create User Pool

Open AWS Cognito from Console > User Pools > Create User Pool

For Provider Types choose: Cognito User Pool

Cognito user pool sign-in options: User name and email

MFA enforcement: no MFA

Required Attributes: name

Configure Message Delivery: Send email with Cognito

App type: Public client 

App client name: cruddur

Client secret: Don't generate a client secret

Review and Create


<img src="https://user-images.githubusercontent.com/66444859/222920891-2b8d1f33-2a04-4bcb-b318-a42b22a81f88.png" width=55%>

### Install AWS Amplify

We will use AWS Amplify SDK Java library for Cognito. 

Install AWS Amplify library
```
cd frontend-react-js
npm i aws-amplify --save
```

If amplify was installed, it will be added to ```package.json```. ```--save``` saves it to ```package.json```.

<img src="https://user-images.githubusercontent.com/66444859/222921206-dd0425d9-192b-4d72-a67e-3f66189ed0c1.png" width=40%>

We need to hook up our cognito pool to our code in the ```App.js```
```
import { Amplify } from 'aws-amplify';

Amplify.configure({
  "AWS_PROJECT_REGION": process.env.REACT_AWS_PROJECT_REGION,
  "aws_cognito_identity_pool_id": process.env.REACT_APP_AWS_COGNITO_IDENTITY_POOL_ID,
  "aws_cognito_region": process.env.REACT_APP_AWS_COGNITO_REGION,
  "aws_user_pools_id": process.env.REACT_APP_AWS_USER_POOLS_ID,
  "aws_user_pools_web_client_id": process.env.REACT_APP_CLIENT_ID,
  "oauth": {},
  Auth: {
    // We are not using an Identity Pool
    // identityPoolId: process.env.REACT_APP_IDENTITY_POOL_ID, // REQUIRED - Amazon Cognito Identity Pool ID
    region: process.env.REACT_AWS_PROJECT_REGION,           // REQUIRED - Amazon Cognito Region
    userPoolId: process.env.REACT_APP_AWS_USER_POOLS_ID,         // OPTIONAL - Amazon Cognito User Pool ID
    userPoolWebClientId: process.env.REACT_APP_AWS_USER_POOLS_WEB_CLIENT_ID,   // OPTIONAL - Amazon Cognito Web Client ID (26-char alphanumeric string)
  }
});
```

Add Env vars to ```docker-compose.yaml```
```
REACT_APP_AWS_PROJECT_REGION: "${AWS_DEFAULT_REGION}"
REACT_APP_AWS_COGNITO_REGION: "${AWS_DEFAULT_REGION}"
REACT_APP_AWS_USER_POOLS_ID: "us-east-1_0fm8fIE3j"
REACT_APP_CLIENT_ID: "app_client_id_from_console"
```

 ### Conditionally show components based on logged in or logged out
 
Inside our ```HomeFeedPage.js```

```
import { Auth } from 'aws-amplify';

// set a state
const [user, setUser] = React.useState(null);

// check if we are authenicated
const checkAuth = async () => {
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

// check when the page loads if we are authenicated
React.useEffect(()=>{
  loadData();
  checkAuth();
}, [])
```

This prevents double API calls:
```
React.useEffect(()=>{
    //prevents double call
    if (dataFetchedRef.current) return;
    dataFetchedRef.current = true;
```
We'll want to pass user to the following components:
```
<DesktopNavigation user={user} active={'home'} setPopped={setPopped} />
<DesktopSidebar user={user} />
```

We'll update ```ProfileInfo.js```
```
import { Auth } from 'aws-amplify';

const signOut = async () => {
  try {
      await Auth.signOut({ global: true });
      window.location.href = "/"
  } catch (error) {
      console.log('error signing out: ', error);
  }
}
```
When tried to open frontend page, we got a blank page. 

<img src="https://user-images.githubusercontent.com/66444859/223570990-1dafe992-f10e-46aa-acb8-ce3ce95b3e6d.png" width=55%>

Opened Inspect and it's showing ```Error: Both UserPoolId and ClientId are required.```

<img src="https://user-images.githubusercontent.com/66444859/223571316-4cfbe7b7-8638-4ac2-b540-cbcf122ca1a3.png" width=40%>

Turns out we had different Envs for ```Auth``` in ```App.js```:
```userPoolWebClientId: process.env.REACT_APP_CLIENT_ID```

Our frontend page is back:

<img src="https://user-images.githubusercontent.com/66444859/223573130-060942d0-5741-450d-84f3-69958741b60d.png" width=55%>

### Signin Page

```
import { Auth } from 'aws-amplify';

const onsubmit = async (event) => {
  setErrors('')
  event.preventDefault();
  try {
    Auth.signIn(username, password)
      .then(user => {
        localStorage.setItem("access_token", user.signInUserSession.accessToken.jwtToken)
        window.location.href = "/"
      })
      .catch(err => { console.log('Error!', err) });
  } catch (error) {
    if (error.code == 'UserNotConfirmedException') {
      window.location.href = "/confirm"
    }
    setErrors('')
  }
  return false
}
```
We tried to sign in and Andrew did not get an error. 

<img src="https://user-images.githubusercontent.com/66444859/223576363-1e4f42ab-a955-48e2-bbae-51cd9ecbf89f.png" width=40%>

Inspect showed this error: ```USR_SRP_AUTH is not enbled for client```. Turns out Andrew chose incorrect App type  ```Other``` instead of ```Piblic client```.

From my side, looking into Inspect I got another error: ```Error! NotAuthorizedException: Incorrect username or password.``` and got that error from UI

<img src="https://user-images.githubusercontent.com/66444859/223593282-adec860d-f718-4f97-b33c-ed42b3a79fd9.png" width=40%>


We missed to add ```preferred_username``` in Required Attributes in User pool, we need to add this later. 

Create a new user in your User Pool

<img src="https://user-images.githubusercontent.com/66444859/223583005-e1809bea-f5eb-40bc-a90d-8258273399c8.png" width=50%>

<img src="https://user-images.githubusercontent.com/66444859/223583317-f2d72c18-6c66-44f9-b5e6-00dc5d324b4e.png" width=55%>


Tried to Sign in, but  it didn't work. Confirmation status is Force change password. We need to Confirm account, but Confirm account button is grey and we can't confirm. And we didn't get Confirmation to specified email address.

Try running AWS CLI command from terminal
```aws cognito-idp admin-set-user-password --username nargizaosmon --password yoursetpassword --user-pool-id us-east-1_CWw2a8NO6 --permanent```

And I don't see ```Force change password``` in Confirmation Status

<img src="https://user-images.githubusercontent.com/66444859/223594032-3d7fce2c-8320-47da-913b-2228a32a58fd.png" width=50%>

But still got error when signing in with new username(email) and password. 

Went back and checked my code, turns out I didn't remove one line in ```SigninPage.js```. Removed the line, recreated my user pool, addedd ```preferred_name``` to requirements and got ```Invalid username or password```, which is correct error. 

<img src="https://user-images.githubusercontent.com/66444859/224200634-9ef1e728-216b-4150-a936-6de6e6d3c731.png" width=35%>

Before force changing password, we were getting ```Cannot read properties of null(reading 'accessToken')``` error, because it was requiring Force Password Change.

<img src="https://user-images.githubusercontent.com/66444859/224200894-bc307ce2-eb1b-4f9c-813f-914d7f23b318.png" width=35%>


Re-created new user, force changed password with AWS CLI command and was able to sign in.

<img src="https://user-images.githubusercontent.com/66444859/224200582-2cc098ae-fc6a-4cbe-8fd8-03ac92dc16eb.png" width=50%>

Sign out worked as well. 

Checked Inspect user messages and it is showing that we were able to sign in.

<img src="https://user-images.githubusercontent.com/66444859/224206099-5ebfa018-0a60-4b34-939a-6b714117e070.png" width=65%>

Since our user name is not set up and showing ```handle``` (as in previous picture), we can add ```preferred_username``` user attribute and it should show up in our page.

<img src="https://user-images.githubusercontent.com/66444859/224206441-2f6fa84b-eb9e-47cb-acc1-8b69650d1792.png" width=45%>

<img src="https://user-images.githubusercontent.com/66444859/224207063-8d970e1a-1edb-4cc6-b51f-7f0887f4e11d.png" width=45%>

### Sign up Page

Add to ```SignupPage.js```

```
import { Auth } from 'aws-amplify';

const onsubmit = async (event) => {
    event.preventDefault();
    setErrors('')
    try {
      const { user } = await Auth.signUp({
        username: email,
        password: password,
        attributes: {
            name: name,
            email: email,
            preferred_username: username,
        },
        autoSignIn: { // optional - enables auto sign in after user is confirmed
            enabled: true,
        }
      });
      console.log(user);
      window.location.href = `/confirm?email=${email}`
    } catch (error) {
        console.log(error);
        setErrors(error.message)
    }
    return false
  }
  ```
  
  ### Confirmation Page
  
  ```
  import { Auth } from 'aws-amplify';
  
  const resend_code = async (event) => {
    setCognitoErrors('')
    try {
      await Auth.resendSignUp(email);
      console.log('code resent successfully');
      setCodeSent(true)
    } catch (err) {
      // does not return a code
      // does cognito always return english
      // for this to be an okay match?
      console.log(err)
      if (err.message == 'Username cannot be empty'){
        setErrors("You need to provide an email in order to send Resend Activiation Code")   
      } else if (err.message == "Username/client id combination not found."){
        setErrors("Email is invalid or cannot be found.")   
      }
    }
  }

  const onsubmit = async (event) => {
    event.preventDefault();
    setErrors('')
    try {
      await Auth.confirmSignUp(email, code);
      window.location.href = "/"
    } catch (error) {
      setErrors(error.message)
    }
    return false
  }
  ```
  
  Tried to sign up and got error ```Username cannot be of email format, since user pool is configured for email alias```.
  
<img src="https://user-images.githubusercontent.com/66444859/224210397-14e8e858-1e32-46f9-a166-5e5394707600.png" width=50%>

  
 We changed ```Cognito user pool sign-in options``` to  ```email``` only by re-creating user pool from console.
 But I changed ``` email: username``` to ```email: email``` in ```SigninPage.js``` to be able to sign in. 
 
 I was able to sign up and got verification code.
 
<img src="https://user-images.githubusercontent.com/66444859/224220235-71a5527a-b2ad-4098-b92d-dfc4f4e8e622.png" width=50%>

<img src="https://user-images.githubusercontent.com/66444859/224220339-6cb0c410-eb5e-4319-a4c5-a97a36055bf4.png" width=30%>

<img src="https://user-images.githubusercontent.com/66444859/224222737-87161290-cfbe-434b-9020-39396293030e.png" width=50%>
  
 Confirmed email
 
<img src="https://user-images.githubusercontent.com/66444859/224222991-95251d76-32ab-48c4-9dec-d6af22604fac.png" width=50%>

### Recovery Page

Add to ```RecoverPage.js```

```
import { Auth } from 'aws-amplify';

const onsubmit_send_code = async (event) => {
  event.preventDefault();
  setErrors('')
  Auth.forgotPassword(username)
  .then((data) => setFormState('confirm_code') )
  .catch((err) => setErrors(err.message) );
  return false
}

  const onsubmit_confirm_code = async (event) => {
    event.preventDefault();
    setErrors('')
    if (password == passwordAgain){
      Auth.forgotPasswordSubmit(username, code, password)
      .then((data) => setFormState('success'))
      .catch((err) => setErrors(err.message) );
    } else {
      setErrors('Passwords do not match')
    }
    return false
  }
```
Signed out, clicked Forgot Password and was able to send recovery email. Password reset code

<img src="https://user-images.githubusercontent.com/66444859/224224324-d4a514fc-53dd-4b00-8363-3a226996ddbf.png" width=30%>

Password has been reset

<img src="https://user-images.githubusercontent.com/66444859/224224534-9b7b84fe-6234-4946-be8f-87e56967ff11.png" width=40%>

### Cognito JWT Server side Verify

Add to ```HomeFeedPage.js```

```
headers: {
  Authorization: `Bearer ${localStorage.getItem("access_token")}`
},
```

Add to ```app.py```

```
import sys

from lib.cognito_jwt_token import CognitoJwtToken, extract_access_token, TokenVerifyError

cognito_jwt_token = CognitoJwtToken(
  user_pool_id=os.getenv("AWS_COGNITO_USER_POOL_ID"), 
  user_pool_client_id=os.getenv("AWS_COGNITO_USER_POOL_CLIENT_ID"),
  region=os.getenv("AWS_DEFAULT_REGION")
)

headers=['Content-Type', 'Authorization'], 
expose_headers='Authorization',

access_token = extract_access_token(request.headers)
try:
  claims = cognito_jwt_token.verify(access_token)
  # authenicatied request
  app.logger.debug("authenicated")
  app.logger.debug(claims)
  app.logger.debug(claims['username'])
  data = HomeActivities.run(cognito_user_id=claims['username'])
except TokenVerifyError as e:
  # unauthenicatied request
  app.logger.debug(e)
  app.logger.debug("unauthenicated")
  data = HomeActivities.run()


````
Add to ```home_activities.py```
```
def run(cognito_user_id=None):

    if cognito_user_id != None:
      extra_crud = {
        'uuid': '248959df-3079-4947-b847-9e0892d1bab4',
        'handle':  'Lore',
        'message': 'My dear brother, it the humans that are the problem',
        'created_at': (now - timedelta(hours=1)).isoformat(),
        'expires_at': (now + timedelta(hours=12)).isoformat(),
        'likes': 1042,
        'replies': []
        }
      results.insert(0,extra_crud)
```

In the ```app.py``` to update CORS:
```
cors = CORS(
  app, 
  resources={r"/api/*": {"origins": origins}},
  headers=['Content-Type', 'Authorization'], 
  expose_headers='Authorization',
  methods="OPTIONS,GET,HEAD,POST"
)
```

Add to ```ProfileInfo.js```
``` localStorage.removeItem("access_token")```



Create new folder with file: ```backend-flask/lib/cognito_jwt_token.py```

```
import time
import requests
from jose import jwk, jwt
from jose.exceptions import JOSEError
from jose.utils import base64url_decode

class FlaskAWSCognitoError(Exception):
  pass

class TokenVerifyError(Exception):
  pass

def extract_access_token(request_headers):
    access_token = None
    auth_header = request_headers.get("Authorization")
    if auth_header and " " in auth_header:
        _, access_token = auth_header.split()
    return access_token

class CognitoJwtToken:
    def __init__(self, user_pool_id, user_pool_client_id, region, request_client=None):
        self.region = region
        if not self.region:
            raise FlaskAWSCognitoError("No AWS region provided")
        self.user_pool_id = user_pool_id
        self.user_pool_client_id = user_pool_client_id
        self.claims = None
        if not request_client:
            self.request_client = requests.get
        else:
            self.request_client = request_client
        self._load_jwk_keys()


    def _load_jwk_keys(self):
        keys_url = f"https://cognito-idp.{self.region}.amazonaws.com/{self.user_pool_id}/.well-known/jwks.json"
        try:
            response = self.request_client(keys_url)
            self.jwk_keys = response.json()["keys"]
        except requests.exceptions.RequestException as e:
            raise FlaskAWSCognitoError(str(e)) from e

    @staticmethod
    def _extract_headers(token):
        try:
            headers = jwt.get_unverified_headers(token)
            return headers
        except JOSEError as e:
            raise TokenVerifyError(str(e)) from e

    def _find_pkey(self, headers):
        kid = headers["kid"]
        # search for the kid in the downloaded public keys
        key_index = -1
        for i in range(len(self.jwk_keys)):
            if kid == self.jwk_keys[i]["kid"]:
                key_index = i
                break
        if key_index == -1:
            raise TokenVerifyError("Public key not found in jwks.json")
        return self.jwk_keys[key_index]

    @staticmethod
    def _verify_signature(token, pkey_data):
        try:
            # construct the public key
            public_key = jwk.construct(pkey_data)
        except JOSEError as e:
            raise TokenVerifyError(str(e)) from e
        # get the last two sections of the token,
        # message and signature (encoded in base64)
        message, encoded_signature = str(token).rsplit(".", 1)
        # decode the signature
        decoded_signature = base64url_decode(encoded_signature.encode("utf-8"))
        # verify the signature
        if not public_key.verify(message.encode("utf8"), decoded_signature):
            raise TokenVerifyError("Signature verification failed")

    @staticmethod
    def _extract_claims(token):
        try:
            claims = jwt.get_unverified_claims(token)
            return claims
        except JOSEError as e:
            raise TokenVerifyError(str(e)) from e

    @staticmethod
    def _check_expiration(claims, current_time):
        if not current_time:
            current_time = time.time()
        if current_time > claims["exp"]:
            raise TokenVerifyError("Token is expired")  # probably another exception

    def _check_audience(self, claims):
        # and the Audience  (use claims['client_id'] if verifying an access token)
        audience = claims["aud"] if "aud" in claims else claims["client_id"]
        if audience != self.user_pool_client_id:
            raise TokenVerifyError("Token was not issued for this audience")

    def verify(self, token, current_time=None):
        """ https://github.com/awslabs/aws-support-tools/blob/master/Cognito/decode-verify-jwt/decode-verify-jwt.py """
        if not token:
            raise TokenVerifyError("No token provided")

        headers = self._extract_headers(token)
        pkey_data = self._find_pkey(headers)
        self._verify_signature(token, pkey_data)

        claims = self._extract_claims(token)
        self._check_expiration(claims, current_time)
        self._check_audience(claims)

        self.claims = claims 
        return claims
```

Add ```Flask-AWSCognito``` to```requirements.txt```.

Add to ```docker-compose.yaml```:
```
AWS_COGNITO_USER_POOL_ID:
AWS_COGNITO_USER_POOL_CLIENT_ID:
```
Was able to Sign in after Cognito Jwt Server side verify

<img src="https://user-images.githubusercontent.com/66444859/224495561-ebd51d68-8456-44d1-a970-31814c1fbd5b.png" width=50%>

Was able to connect to Backend page 

<img src="https://user-images.githubusercontent.com/66444859/224495668-648fb835-d3cb-4c5b-85b2-b2a40a388246.png" width=45%>


#### Reference
[Amplify docs](https://docs.amplify.aws/lib/auth/getting-started/q/platform/js/)
