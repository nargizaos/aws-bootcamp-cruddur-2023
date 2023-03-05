# Week 3 — Decentralized Authentication

### Create User Pool

Open Cognito from AWS Console > User Pools > Create User Pool

<img src="https://user-images.githubusercontent.com/66444859/222920891-2b8d1f33-2a04-4bcb-b318-a42b22a81f88.png" width=55%>

### Install AWS Amplify
```npm i aws-amplify --save```

If amplify was installed, it will be added to ```package.json```

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
 ### Conditionally show components based on logged in or logged out


#### Reference
(Amplify docs)[https://docs.amplify.aws/lib/auth/getting-started/q/platform/js/]
