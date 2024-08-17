---
layout: post
title: Set cookies in Lambda function triggered by API Gateway for a different subdomain
date: 2024-08-19 10:00:00 -0500
description: Learn how to set cookies from a lambda function that was triggered by API Gateway using a subdomain.
img: posts_imgs/apigw-lambda-set-cookies/apigw_lambda.jpeg
tags: [python, AWS, API gateway, lambda, jwt, sign out]
---

In this quick post we will review how to set cookies to a specific value from a different subdomain using Lambda service triggered by API gateway. For this post, we will develop a sign out process where the API Gateway has a GET route https://subdomain.domain.com/logout which can be triggered to delete the cookies (which contain JWT for authroization) and redirect to sign in page.

The website is located in https://subdomain1.domain.com and the API gateway used the custom domain https://subdomain2.domain.com. It is important that the domain is the same for both cases.

In the main page we have an anchor tag for sign out with a javascript on click event to send the request to API Gateway

```html
<a id="signOut" href="#" onclick="logout()" title="Signout"><li>Sign Out</li></a>
```

The function triggered is a havascript function as below:


```javascript
// Get token from cookie
function get_token() {
  return document.cookie.replace(/(?:(?:^|.*;\s*)token\s*\=\s*([^;]*).*$)|^.*$/, "$1");
}
// Logout
function logout() {
  fetch('https://subdomain.domain.com/logout', {
    headers: {
        'Content-Type': 'application/json',
        'Authorization': get_token()
    },
    credentials: 'include'
  })
    .then(response => {
      window.location.href = "<LOGIN_PAGE>";
  })  
}
```

The function get_token() help us to recover the JWT token used to add it as Authroization header. This is required to pass the "Method Request" stage from the API gateway GET /logout route which uses a Cognito authorizer to verify the token and confirm the user is authorized to perform the action.

![Method request configuration](/assets/img/posts_imgs/apigw-lambda-set-cookies/method_request.png)

In the stage of "Integration Request" we use a Lambda integration with "Lambda proxy integration" enabled, this way the API Gateway directly passes the incoming request from the client as an event object to the Lambda function.

Finally we need to configure the CORS configuration for the preflight stage (OPTIONS method). The "Access-Control-Allow-Origin" should be the subdomain used by the main page (not the subdomain of the API gateway custom domain)

![API Gateway CORS Config.](/assets/img/posts_imgs/apigw-lambda-set-cookies/apigw_cors.png)

The lambda function triggered by API gateway is now capable to set the value for the tokens in the cookies. The cookies are available for the full domain as they are set with "Domain=domain.com". You can also set only for the subdomain, but in our case we prefer to keep them available for any subdomain inside domain.com

```python
import json

def lambda_handler(event, context):
    response = {
        'statusCode': 200,
        "isBase64Encoded": True,
        'headers': {
            'Access-Control-Allow-Origin': 'https://subdomain.domain.com',
            'access-control-expose-headers': 'Set-Cookie',
            'Access-Control-Allow-Credentials': True,
            "Access-Control-Allow-Headers": "*"
        },
        'multiValueHeaders': {"Set-Cookie": ['token1=deleted; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT; Domain=domain.com; Secure;', 'token2=deleted; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT; Domain=domain.com; Secure;']}
    }
    return json.loads(json.dumps(response, default=str)) 
```

The previous code is removing the existing cookies and the browser function logout() can redirect the user to the login page, since the cookies are removed then the user needs to authenticate again to access.

And that's it. Now you can managed the cookies of your website from a Lambda function triggered by API gateway when using multiple subdomains.