
# Messaging Extension Flow

1. Post request received in controller
Request has a Bearer Token in the Header

The token is an OpenID token
```json
{
  "alg": "RS256",
  "kid": "ZyGh1GbBL1xd1kOxEYchc1VPSQQ",
  "typ": "JWT",
  "x5t": "ZyGh1GbBL1xd1kOxEYchc1VPSQQ"
}.{
  "serviceurl": "https://smba.trafficmanager.net/amer/",
  "nbf": 1645135040,
  "exp": 1645138640,
  "iss": "https://api.botframework.com",
  "aud": "ee70da8a-1222-4916-bf83-e7ad34327b25" // Azure Bot AAD App ID
}.[Signature]
```

- More detail
Open ID Connect token is validated in the Bot Framework Adapter

https://login.botframework.com/v1/.well-known/openidconfiguration
```json
{
  "issuer": "https://api.botframework.com",
  "authorization_endpoint": "https://invalid.botframework.com",
  "jwks_uri": "https://login.botframework.com/v1/.well-known/keys",
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "token_endpoint_auth_methods_supported": [
    "private_key_jwt"
  ]
}
```

## The turnContext.Activity object contains this data
```json
{
    "type": "invoke",
    "id": "f:e50c0694-e383-94ff-7197-687e94cecec6",
    "timestamp": "2022-02-18T14:49:26.847+00:00",
    "localTimestamp": "2022-02-18T09:49:26.847-05:00",
    "localTimezone": "America/New_York",
    "serviceUrl": "https://smba.trafficmanager.net/amer/",
    "channelId": "msteams",
    "from": {
        "id": "29:17GtpxGgHIa1f4tdko79rRk-_e6NHwlG8RKqI3ow010CQz8ZFhdkhF0SEDc8glS7Qle7QjTlWlGhW6RMykmq8wg",
        "name": "Brennen Cage",
        "aadObjectId": "8324bcf9-6ce3-32zj-a146-b69e11ccbc45",
        "role": null
    },
    "conversation": {
        "isGroup": null,
        "conversationType": "personal",
        "id": "a:1EsSe9tfG2hbfQK4Q-S4um1TVIdfConk3x2Bj8jXBwdq1QukJQcy7N9SWxEUfq6XE-IcV1PQ7P_nM2QV3vzPKOomAJuVREQvSj0ssluyscIrzEo0elqpOEnzU8iWfPM-6",
        "name": null,
        "aadObjectId": null,
        "role": null,
        "tenantId": "9bc3e81f-99fc-4a04-b5ec-06eb59a03d44"
    },
    "recipient": {
        "id": "28:ee70da8a-1222-4916-bf83-e7ad34327b25",
        "name": "MsgExtensionAzureBot",
        "aadObjectId": null,
        "role": null
    },
    "textFormat": null,
    "attachmentLayout": null,
    "membersAdded": null,
    "membersRemoved": null,
    "reactionsAdded": null,
    "reactionsRemoved": null,
    "topicName": null,
    "historyDisclosed": null,
    "locale": "en-US",
    "text": null,
    "speak": null,
    "inputHint": null,
    "summary": null,
    "suggestedActions": null,
    "attachments": null,
    "entities": [
        {
            "type": "clientInfo",
            "locale": "en-US",
            "country": "US",
            "platform": "Web",
            "timezone": "America/New_York"
        }
    ],
    "channelData": {
        "tenant": {
            "id": "9bc3e81f-99fc-4a04-b5ec-06eb59a03d44"
        },
        "source": {
            "name": "compose"
        }
    },
    "action": null,
    "replyToId": null,
    "label": null,
    "valueType": null,
    "value": {
        "commandId": "searchQuery",
        "parameters": [
            {
                "name": "initialRun", // or could be searchQuery
                "value": "true"       // "my search value"
            }
        ],
        "queryOptions": {
            "count": 25,
            "skip": 0
        }
    },
    "name": "composeExtension/query",
    "relatesTo": null,
    "code": null,
    "expiration": null,
    "importance": null,
    "deliveryMode": null,
    "listenFor": null,
    "textHighlights": null,
    "semanticAction": null,
    "callerId": "urn:botframework:azure"
}
```

## Now we need to get a token for the user

```c#
var tokenResponse = await GetTokenResponse(turnContext, action.State, cancellationToken); //State = null
```

_connectionName = OAuth Connection from Azure Bot
magicCode = State = null
```c#
var tokenResponse = await provider.GetUserTokenAsync(turnContext, _connectionName, magicCode, cancellationToken: cancellationToken); 
```

An OAuth client is created using the Azure Bot Client ID and Client Secret - refer to bottom

token = https://api.botframework.com/api/usertoken/GetToken?userId=29%3A17GtpxGgHIa1f4tdko79rRk-_e6NHwlG8RKqI3ow010CQz8ZFhdkhF0SEDc8glS7Qle7QjTlWlGhW6RMykmq8wg&connectionName=GoogleG1&channelId=msteams&code=
token = null => Generate a sign in Url and return to user
token = !null => use token for downstream requests

## sign in url

```c#
var signInLink = await (turnContext.Adapter as IUserTokenProvider).GetOauthSignInLinkAsync(turnContext, _connectionName, cancellationToken);
```
--client cred auth token is sent
https://api.botframework.com/api/botsignin/GetSignInUrl?state=eyJjb25uZWN0aW9uTmFtZSI6Ikdvb2dsZUcxIiwiY29udmVyc2F0aW9uIjp7...

oAuthEndpoint = https://login.microsoftonline.com/botframework.com
scope = https://api.botframework.com

this state code = this base64Encoded
```json
{
  "connectionName": "GoogleG1",
  "conversation": {
    "activityId": "f:21ea8a22-4a5b-3c22-2155-d09fb53127fd",
    "user": {
      "id": "29:17GtpxGgHIa1f4tdko79rRk-_e6NHwlG8RKqI3ow010CQz8ZFhdkhF0SEDc8glS7Qle7QjTlWlGhW6RMykmq8wg",
      "name": "Brennen Cage",
      "aadObjectId": "8324bcf9-6ce3-32zj-a146-b69e11ccbc45",
      "role": null
    },
    "bot": {
      "id": "28:ee70da8a-1222-4916-bf83-e7ad34327b25",
      "name": "MsgExtensionAzureBot",
      "aadObjectId": null,
      "role": null
    },
    "conversation": {
      "isGroup": null,
      "conversationType": "personal",
      "id": "a:1EsSe9tfG2hbfQK4Q-S4um1TVIdfConk3x2Bj8jXBwdq1QukJQcy7N9SWxEUfq6XE-IcV1PQ7P_nM2QV3vzPKOomAJuVREQvSj0ssluyscIrzEo0elqpOEnzU8iWfPM-6",
      "name": null,
      "aadObjectId": null,
      "role": null,
      "tenantId": "9bc3e81f-99fc-4a04-b5ec-06eb59a03d44"
    },
    "channelId": "msteams",
    "locale": "en-US",
    "serviceUrl": "https://smba.trafficmanager.net/amer/"
  },
  "relatesTo": null,
  "botUrl": null,
  "msAppId": "ee70da8a-1222-4916-bf83-e7ad34327b25"
}
```

that retuns = https://token.botframework.com/api/oauth/signin?signin=f7e5543c902f441796859304743c04b3

this is the link the user clicks on to do the auth

That request returns = 
https://token.botframework.com/.auth/web/login/367ee542-a234-d782-cff9-814c00300d37_32eda70f-213a-d61f-bf6a?redirect_uri=https://token.botframework.com/api/oauth/PostSignInCallback?signin=fcaada30162f49b08466beaed0e6370b

Get request =
https://token.botframework.com/.auth/web/login/367ee542-a234-d782-cff9-814c00300d37_32eda70f-213a-d61f-bf6a?redirect_uri=https://token.botframework.com/api/oauth/PostSignInCallback?signin=fcaada30162f49b08466beaed0e6370b

Returns
https://accounts.google.com/o/oauth2/v2/auth?client_id=259923230032-ahlv7c0en40b22p68kal0a94r3dseg28.apps.googleusercontent.com&response_type=code&redirect_uri=https://token.botframework.com/.auth/web/redirect&scope=openid https://www.googleapis.com/auth/userinfo.email&state=8991fa390d8c4de6b93e24184fbb4759


Get request
https://accounts.google.com/o/oauth2/v2/auth?client_id=259923230032-ahlv7c0en40b22p68kal0a94r3dseg28.apps.googleusercontent.com&response_type=code&redirect_uri=https://token.botframework.com/.auth/web/redirect&scope=openid https://www.googleapis.com/auth/userinfo.email&state=8991fa390d8c4de6b93e24184fbb4759

Returns redirect 
https://token.botframework.com/.auth/web/redirect?state=8991fa390d8c4de6b93e24184fbb4759&code=4/0AX4XfWhWIu14VRnqCYRRWVK6GR3FwVMeJ9Iy2TxXLoEoZSfvAn05camQTRDAGFrIxflrww&scope=email https://www.googleapis.com/auth/userinfo.email openid&authuser=0&prompt=none

Get request 
https://token.botframework.com/.auth/web/redirect?state=8991fa390d8c4de6b93e24184fbb4759&code=4/0AX4XfWhWIu14VRnqCYRRWVK6GR3FwVMeJ9Iy2TxXLoEoZSfvAn05camQTRDAGFrIxflrww&scope=email https://www.googleapis.com/auth/userinfo.email openid&authuser=0&prompt=none

Returns moved
https://token.botframework.com/api/oauth/PostSignInCallback?signin=fcaada30162f49b08466beaed0e6370b&code=609f02ce8dd3468fb8717340b1cd40a5

Get Request
https://token.botframework.com/api/oauth/PostSignInCallback?signin=fcaada30162f49b08466beaed0e6370b&code=609f02ce8dd3468fb8717340b1cd40a5

Loads a page with a code

microsoftTeams.initialize();
microsoftTeams.authentication.notifySuccess('137249');

This code is passed to the invoke command and passed in the get token call code=137249




Client Credentials bot to Azure

https://login.windows.net/botframework.com/
with client credentials

returns auth token and this is sent as a header to other requests

authtoken = eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dC....
```json
{
  "typ": "JWT",
  "alg": "RS256",
  "x5t": "Mr5-AUibfBii7Nd1jBebaxboXW0",
  "kid": "Mr5-AUibfBii7Nd1jBebaxboXW0"
}.{
  "aud": "https://api.botframework.com",
  "iss": "https://sts.windows.net/d6d49420-f39b-4df7-a1dc-d59a935871db/",
  "iat": 1645204765,
  "nbf": 1645204765,
  "exp": 1645291465,
  "aio": "E2ZgYPCrtD8zzb5FeMM3J23uWZOdAA==",
  "appid": "ee70da8a-1222-4916-bf83-e7ad34327b25",
  "appidacr": "1",
  "idp": "https://sts.windows.net/d6d49420-f39b-4df7-a1dc-d59a935871db/",
  "rh": "0.AW4AIJTU1pvz902h3NWak1hx20IzLY0pz1lJlXcODq-9FrxuAAA.",
  "tid": "d6d49420-f39b-4df7-a1dc-d59a935871db",
  "uti": "qneDnY7h-0CkzoNyvRgtAA",
  "ver": "1.0"
}.[Signature]
```

