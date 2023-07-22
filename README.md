# AWS Serverless REST APIs

## Basics

### Introduction to REST API
- REST APIs should stateless, and HTTP requests do not keep and do not share transient information b/w each other.
- REST API Resources & URIs 
    - Create a new Resource (REST Style)
        - Resource URIs should be based on nouns, resource and not on verbs and it is always plural.
        - For example, to create a new user, we should use /users and not /createUser
      - Get Resource (REST style)
        - To get details, we will start with resource name that is plural like users and then we would add forward slash and then specify the id of the resource.
        - For example, /users/123 and not /usersDetails?id=123
    
### Lambda function
Services that invoke Lambda function synchronously -
- API Gateway
- Amazon Cognito
- ELB (Application LB)

Services that invoke Lambda function asynchronously -
- Amazon S3
- Amazon SNS
- Amazon CW Logs

Services that Lambda read events from -
- Amazon DDB
- Amazon SQS
- Amazon Kinesis
- Amazon MQ

There are more, just captured a few examples.

#### Lambda execution environment lifecycle
- It has 3 phases:
    - Init: Extension Init, Runtime Init (like JVM) and Function Init (Download source code and Dependency/App startup).
    - Invoke
    - Shutdown: Runtime shutdown and Extension shutdown. If there are no more requests to cater then Lambda function would go into shutdown phase.

The time to initialise the lambda execution environment is called cold start duration.

#### Cold start, warm start, & provisioned concurrency
- There are 2 components to cold start:
    - Download code + Start new execution environment = AWS related Lambda cold start
    - Invoke code outside the handler() = Code Initialisation like init db connections, init objects, loading frameworks like springboot, etc.

- When there are no requests to invoke Lambda for a certain amount of time, Lambda service will freeze the execution environment, it will not destroy it right away.
- The length of environment's lifetime is influenced by various factors that are not configurable by developer today. So to improve resource management and performance, Lambda service retains the execution environment for a non-deterministic period of time.
- We can use provisioned concurrency to reduce cold start, when we enable provisioned concurrency, Lambda created certain number of execution environments in advance (as configured).

### AWS SAM
- AWS SAM = AWS Serverless Application Model
- Framework to build and deploy serverless apps to AWS.
- template.yaml file will be created for every serverless application created using SAM.
    - This file will describe the serverless app and the resources it uses as per SAM specification.
- During the deployment, SAM will transform the instructions in the template.yaml file in AWS CFT and CFT will then create and configure resources for the application.
- For Lambda -
    - https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-function.html
    - EventSource property of serverless function: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-function-eventsource.html
    - For example if the EventSource type is API then SAM framework would create a new API project in API Gateway and it would configure a resource there like '/users' with http post method.

### Creating a new IAM User
- When creating an IAM user, we can choose how the user will access AWS.
- To perform programmatic access, we will choose that option and we will be provided with access key id and secret access key for AWS CLI, API, SDK and other development tools.

Note: Google gson has been used to convert object to/from json string.

## AWS API Gateway

### Introduction
We can create API on API Gateway console, when creating API we have to mention:
- Protocol: REST API or WebSocket API.
- Create new API: New API or Clone from existing API or Import from Swagger/OpenAPI3 or Example API.
- Settings:
    - Endpoint Type: Regional or Edge Optimised or Private.
        - Edge Optimised will make your API deployed in CloudFront network, the API request would be routed to the nearest CloudFront point of presence.
    
### Method request overview
Flow is:
- Method Request -> Integration Request -> Target
- Method Response <- Integration Response <- Target

For Method Request, we have/control -
- Authorization - NONE OR AWS IAM OR Choose Cognito/Lambda Authorizer
- Request Validator - NONE OR Validate body OR Validate body, query string parameters, and headers OR Validate query string parameters, and headers.
- API Key Required - True OR False.

We can also specify the -
- URL query string parameters that the API can accept and are required or optional
- HTTP request headers that the API can accept and are required or optional.
- Request body - its model that the request needs to follow to be valid.

If the request to API will not conform to this then the request will be rejected and not pass to the target backend.

### Integration request overview
- Allows us to configure which service should be used as target backend for this API endpoint and how should request be transformed before it is send to target backend.
- We can also specify if any mapping needs to be done on the method request for the request body, headers, path or query variables.

### Integration Response
- API Gateway can intercept the response sent by backend service and create a new response body payload.

### Method Response
- We can configure what response body model and headers this API endpoint can return.

Note -
- The request is first sent to API Gateway, which validates the request, authnz, etc such as charon and then the request is forwarded to target backend. 
- This target backend can be a LB, which further sends request to app instances, LB can be AWS ELB which forwards requests to ECS tasks or k8s service which forwards request to k8s pods.

## Creating Mock API Endpoints

### Path Parameter
- OPTIONS HTTP Method is created to support preflight request from coming from browser, this is needed for CORS.

### Deploying Mock API
- To deploy the API and make it public, we choose Deploy API option and create new/choose some existing stage name.

### Reading Query String Parameter in a Mapping Template
- We can use `$input.params('fieldName')` in the mapping template of integration response to use input/request parameters in the response.

### Path Parameter
- Path variable can be created for a route by creating resource under a resource (route) i.e. sub-resource. The value must be in curly braces for sub-resource like: {userId}.

### Export API and Test with Swagger
- Swagger is a tool for implementing the OpenAPI specification.
- In order to export the API from API Gateway as an OpenAPI3 file (and import into a swagger tool) we need to deploy the API first.

## Validating HTTP Request

### Validating Request Parameters & Headers
- You can configure API Gateway to perform basic validation of an API request before proceeding with the integration request. When validation fails, API Gateway immediately fails the request, returns a 400 error response to the caller, and publishes the validation results in the CloudWatch Logs.
- The required request parameters in the URI, query string and headers of an incoming request are included and non-blank.
- The applicable request payload adheres to the configured JSON schema request model of the method.
- In other to validate the request, we need to choose validate options in the request validator setting of method request.

### Validating Request Body
- To validate the request body, we need to create a model.

## Data Transformations
- Data Transformation via API Gateway can be done on:
    - Incoming JSON from HTTP client
    - Outgoing JSON from Lambda/target backend
- https://github.com/simplyi/DataTransformationExample/

## Lambda as Proxy
- If the input parameter of handleRequest method is APIGatewayProxyRequestEvent then it is for the case when API Gateway is used as a proxy for Lambda.
- If we want to configure API gateway to modify request and response payloads, we need to disable the proxy option on the API gateway side.
- Because this proxy option would now be disabled on API gateway side, Lambda function will no longer receive an object of APIGatewayProxyRequestEvent datatype.
- We can now (when proxy is disabled on gateway side) mention a custom object like Map as the input parameter type, the framework/library will try to convert the json into this type of Java object.
- Both input and output can be of Map<String, String> type when we have disabled the proxy integration for Lambda.

## Debug Lambda function locally
- A docker image is created for the Lambda and uploaded to Amazon ECR by SAM. This docker image is then pulled and run locally in a docker container when using the below command:
```
sam local invoke <Lambda-Function-Name> --event <Event-FileName>
```
- Now to debug Lambda locally, we attach a debugger using -d flag to the above command like below:
```
sam local invoke <Lambda-Function-Name> --event <Event-FileName> -d 5858
```
- We create a JVM Remote Debug configuration in IDE (where code is present) and attach a debugger to 5858 port and run this configuration in debug mode.
- Now the debugger present in IDE is started on port 5858 where SAM attaches the Lambda to and triggers Lambda with the event supplied.

P.S. Docker container (Object) is a running instance of docker image (Class).

## Error Responses
- https://github.com/simplyi/ErrorResponseExample
- The client application will get the http default 5xx response only if the exception is not handled (using try/catch) in the java code.
- But since we are handling exception in our java code, we can set custom http status code and a custom json.
- https://github.com/simplyi/ErrorResponseExampleNonProxy

## Lambda Function Versions
- We can configure API endpoint to use any of the published lambda versions.
- We can only change code and configuration in the lambda function that is labeled as latest. Those functions that were published as version 1 or 2 become immutable and cannot be changed anymore.
- We can choose 'Publish New Version' option on AWS Lambda console, this would publish a new version of Lambda function by taking snapshot of current changes and configuration.
- The Function ARN for version 1 will have :1 in the end, version 2 will have :2 in the end. If we don't mention any :number like :1 then the Latest Lambda code would be used.

### Creating an Alias
- We can create function alias on AWS Lambda console and point that alias to a particular Lambda version(s) like 60% of v1 and 40% of v2.
- Each alias will have a name like it can be prod. We can mention the Lambda ARN, with this alias (:prod) in the API endpoint (target backend) in the integration request section of API gateway project.

### Promoting and Deleting Canary
- If we create canary for lets say prod stage of an api gateway project and if we deploy any changes to prod stage then first those changes will be deployed to canary (could be configured as 5%) and till the time we don't promote the canary, the new changes still won't be completely deployed to prod stage (only 5% of traffic will run on new changes via canary).
- FYI - Each stage (or deployment stage) in an API gateway project has an endpoint, using which the requests can be send to API gateway.

## Cognito User Pool
- Cognito allows us to implement the functionality of user authentication, authorization as well as user management.
- The 2 main components of Amazon Cognito are User Pools and Identity Pools.
- User pools are user directories that provide signup and sign-in options for your API users. And Identity pools enable you to grant user access to other AWS services.

## Creating User Pool
- How do you want end users to sign in?
    - Email address or phone number, users can choose an email or phone as their "username" to sign up & sign in.
    - Allow email addresses. The users can now sign up or sign in using their email as username.
    - Users sign-in & attribute options can't be changed once the user pool is created.
- Which standard attributes you want to require for signup?
    - Additionally to the email address, we can require a user to provide their name, gender, etc. We can add custom attribute as well.
- What password strength you want to require?
- Do you want to allow users to sign themselves up?
- Do you want to enable MFA?
- How will a user be able to recover their account?
- Do you want to customise your email address? (From which verification email would be send)
- Do you want to send email  through Amazon SES configuration?
- Do you want to customise your email verification message?
- Which app clients will have access to this user pool?
    - Covered more in below section for this question.

## Creating an App Client
- We will need an app client to be able to access this user pool from our Lambda function. These app clients will be given unique id and an optional secret key to access this user pool.
- The refresh token is used to refresh the access and id token. The access token will be used to access protected api endpoints and id token is a json web token that contains claims about the identity of the authenticated user such as name, email and phone number.
- If these token are incorrect or expired then our lambda function will not be invoked.

### SignUp
- In the SignUpRequest sent to AWS Cognito to User Pool, we need to provide the clientId and secretKey along with the new client details like username and password for registration.

Note -
- We can encrypt the Lambda environment variables (using a key from AWS KMS) and mention same encryption key on Lambda console for decryption.
- In the Lambda Java code, we would need to decrypt these values as well.

### Confirm User Account
- After sending InitiateAuthRequest to Cognito for a user, in response, we send different tokens which are IdToken, AccessToken and RefreshToken.
- These tokens can be used in other requests to access resources.

### Add User to Group
- A group name in User Pool in Cognito is assigned an IAM role which determines what operations can members of this group do, such as adding data to DDB.

## Cognito Authorizer
- We can secure API endpoint with Cognito authorizer and access these endpoint using user's IdToken and AccessToken.
- ID Token and Access Token are different types of JSON web tokens (JWT) that contain claims about the authenticated user or the authorized API operations.
- ID Token is sent to the client application as part of an OpenID Connect flow and is used by the client to authenticate the user. The ID Token contains claims about the identity of the user, such as name, email, and phone_number. This token is passed in Authorization HTTP header.
- Access Token is used to authorize API operations in the context of the user in the user pool. The Access Token contains scopes and groups to authorize access to your resources.
- Important -
    - When Cognito Authorizer is integrated with API gateway for some resources (like /users), we need to provide Authorization HTTP header with the value of ID token. 
    - Access Token header also needs to be provided (via header or body) if its value is to be needed to perform some API operation like getting some user details.

## Lambda Authorizer
- We can implement a custom lambda authorizer to perform any custom authorisation, this lambda would return a policy document to api gateway telling what to do i.e. whether to accept or deny the request. This policy can be cached for sometime by api gateway.

### Creating Lambda Authorizer
- Lambda Authorizer is also known as Custom Authorizer, and is an API Gateway feature that uses Lambda function to control access to APIs.
- It is useful when we want to implement custom authorization for your API endpoint. For example, to validate and decode the JWT token, we can do validation in the Lambda and response to API Gateway whether the request should be denied or allowed.
- The Lambda will have access to the request details such as 'Authorization' header and userId from path parameters, and we will use it to decide whether we want to allow this request to pass through or deny.
- If the validation is success then a policy document would be returned by Lambda Authorizer to API gateway which it would use to approve or deny the request (based on authorization access of user), else if the validation is failure then Lambda would tell API gateway to deny the request straightaway.
- We can enable caching for policy at API gateway end.

### Generating Policy Document
- The Lambda Authorizer needs to return an object having 2 fields -
    - principleId (which is the username) and,
    - a policy which consists of a list of statements that tells whether access is allow/deny for certain resources.

### Implement JWT Validation function
- To be able to validate the authorization token that was issued by AWS Cognito, we will need to use the RSAPublicKey Java object.
- We can create an instance of this object if we extract a corresponding public key for our User Pool.
- The public key is available as a part of the JSON Web Key Set and is located at a specific URL that you can open and preview in a browser window. You can locate it at https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json
- The public key we will extract from the .well-known/jwks.json endpoint will be used to verify the RS256 signature of the Authorization JWT Token.

Note -
- We have verified the RS256 signature of token by verifying its signature using RSAPublicKey issued by AWS Cognito and also verified few of its claims such as subject, audience, etc.
- Subject should be the userName which is Cognito given user id and audience will be app client id which we use to access AWS Cognito from Lambda app.

### Add Policy to Lambda Authorizer function
- In Lambda's Policies section, we should mention the action that Lambda is allowed to do like get user details from Cognito, read from SQS, etc. and the resource on which Lambda can execute these actions.
