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
When tried to open frontend page, we got a blan page. 
<img src="https://user-images.githubusercontent.com/66444859/223570990-1dafe992-f10e-46aa-acb8-ce3ce95b3e6d.png" width=55%>

Opened Inspect and it's showing ```Error: Both UserPoolId and ClientId are required.```

<img src="https://user-images.githubusercontent.com/66444859/223571316-4cfbe7b7-8638-4ac2-b540-cbcf122ca1a3.png" width=40%>

Turns out we had different Envs for ```Auth``` in ```App.js```:
```userPoolWebClientId: process.env.REACT_APP_CLIENT_ID```

Our frontend page is back:

<img src="https://user-images.githubusercontent.com/66444859/223573130-060942d0-5741-450d-84f3-69958741b60d.png" width=55%>

##### Signin Page

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

Inspect showed this error: ``USR_SRP_AUTH is not enbled for client```. Turns out Andrew chose incorrect App type ```Other``` instead of ```Piblic client```.

From my side I got another error: ```Error! NotAuthorizedException: Incorrect username or password.```

We missed to add ```preferred_username``` in Required Attributes in User pool, we need to add this later. 

Create a new user in your User Pool

<img src="https://user-images.githubusercontent.com/66444859/223583005-e1809bea-f5eb-40bc-a90d-8258273399c8.png" width=50%>

<img src="https://user-images.githubusercontent.com/66444859/223583317-f2d72c18-6c66-44f9-b5e6-00dc5d324b4e.png" width=55%>


Tried to Sign in, but  it didn't work. Confirmation status is Force change password. Since we need to Confirm account, but Confirm account button is grey and we can't confirm. And we didn't get Confirmation to specified email address.



#### Reference
[Amplify docs](https://docs.amplify.aws/lib/auth/getting-started/q/platform/js/)
