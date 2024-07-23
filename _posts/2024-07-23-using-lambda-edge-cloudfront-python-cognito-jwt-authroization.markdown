---
layout: post
title: Using Lambda@Edge and CloudFront with Python for JWT credentials exchange and token verification with Cognito
date: 2024-07-11 10:00:00 -0500
description: Learn how to implement Cognito JWT credentials exchange and token validation from CloudFront/Lambda@Edge using Python
img: posts_imgs/lambdaedge-cloudfront-cognito-auth/auth-flow.png
tags: [python, AWS, cloudfront, lambda, cognito, jwt, auth]
---

Using Lambda@Edge with Python to exchange JWT credentials or validate tokens sounds a common task to implement, however it is not easy to find examples over the internet. Using javascript you will find multiple solutions but for Python you won't be so lucky. And the fact that Lambda@Edge package must be maximum 1MB in size makes the tasks even harder.

So I have developed a solution using many different public posts as a base, but the most tricky part was the package size limitation. For this case I had to edit existing libraries to only keep the necessary modules. I have [attached](/assets/files/lambda-edge-cloudfront-package/Archive.zip) the libraries ready to use with your code so you won't have to deal with this problem. It also contains the code I have used in this post with cognito and cloudfront.

Typical flow to perform auth with Cognito from Lambda and Cloudfront
![Lambda@Edge auth with Cloufront and Cognito](/assets/img/posts_imgs/lambdaedge-cloudfront-cognito-auth/auth-flow.png)

Now let's understand more about the problem and the solution implemented.

Basically the problem we may face is when designing a serverless application in AWS, we probably need to use the following services:

-   AWS Cognito user pools to handle sign in and sign up.
-   AWS CloudFront as CDN for our static files.
-   AWS S3 as object storage for the static files (like HTML, CSS, Javascript files or even full client side application like React)
-   AWS Lambda to perform the backend business logic.
-   AWS RDS / DynamoDB / DocumentDB etc. for database.

Implementing a signup/signin process can be done very fast with Cognito. The service will handle the user creation, authentication, recovery password, email verification etc. Usually, when working with Cognito, the user sign in with its credential (or federate through external providers like google or facebook) and if successfull, Cognito provides a code in the query string that we can use from the backed to exchange for tokens (JWT).

The tokens can be then store as cookies so the user does not have to authenticate continously. The solution we are proposing here focuses on the process after the authentication has been done with Cognito. Basically the cognito will redirect the user to the Allowed callback URLs, at this point is where Cloudfront, S3 and Lambda (Lambda@Edge to be more precise) come into play.

In Cloudfront we can include all the static files in S3 as origins and configure a "viewer-request" event which trigers a Lambda@Edge function which is basically a Lambda function that runs in Edge locations for lower latency.

![Lambda@Edge trigger by events in Cloudfront](/assets/img/posts_imgs/lambdaedge-cloudfront-cognito-auth/lambdaedge_cloudfront.png)

As the image above shows, the "viewer-request" event is triggered when the behavior configured in CloudFront happen and this behavior is configured with the "viewer-request" event.

In our case, the "viewer-request" event triggers our Lambda function. Let's review the python code used by the Lambda function.

The "lambda_handler" is the main function triggered when the lambda is invoked. This main function executes 3 possible block codes:

1. User is trying to log in. For this case the query string in the URL contains a parameter "code" which is the code returned from Cognito when the log in succeed.

```python
    referer = headers['referer'][0]['value']
    code = request['querystring'].split('code=')[1]
    tokens = get_tokens(code)
    access_token_verification = verify_token_signature(tokens['access_token'])
    id_token_verification = verify_token_signature(tokens['id_token'])
    refresh_token_verification = verify_token_signature(tokens['refresh_token'])
```

Using the "code" parameter, we can now exchange it for tokens (JWT), for this we use the following function:

```python
    def get_tokens(code):
        headers = {
            'Content-Type' : 'application/x-www-form-urlencoded'
        }
        url = f"{AUTH_URL}oauth2/token?grant_type=authorization_code&code={code}&client_id={COGNITO_CLIENT_ID}&redirect_uri={BASE_URL}"
        response = http.request('POST', url, headers=headers)
        return json.loads(response.data.decode('utf-8'))
```

Then the code just checks whether the tokens are valid and if they are them we proceed to store the tokens as cookies.

2. The request includes cookies with tokens, so in this code block we validate the cookies before granting access.

```python
    cookie.load(headers['cookie'][0]['value'])
    cookies = {}
    for key, val in cookie.items():
        cookies[key] = val.value
    print(f"Cookies: {cookies}")
    access_token_verification = verify_token_signature(cookies['access'])
    id_token_verification = verify_token_signature(cookies['id'])
    refresh_token_verification = verify_token_signature(cookies['refresh'])
```

The function "verify_token_signature" takes a token (access, id or refresh) and verifies its authenticity.

```python
    def verify_token_signature(token):
        for idx, k in enumerate(token_signing_keys):
            pub_key = jwk.construct(k)
            message, encoded_sig = token.rsplit('.', 1)
            decoded_sig = base64url_decode(encoded_sig.encode())
            try:
                pub_key.verify(message, decoded_sig)
                return {
                    'status': 200,
                    'pub_key': pub_key
                }
            except:
                if len(token_signing_keys) == idx+1:
                    return {
                        'status': 401
                    }
                else:
                    pass
```

If the tokens are valid, then the request is returned without modification, otherwise the cookies are cleared and the user is redirected to the login page (Cognito login URL).

3. At last, if there is no code in the URL query string and no cookies with tokens, then the user is redirected to the login page.

```python
    response = {
            'status': '302',
            'statusDescription': 'Redirect',
            'headers': {
                'location': [{
                    'key': 'Location',
                    'value': AUTH_URL+'login?client_id='+COGNITO_CLIENT_ID+'&response_type=code&scope=email+openid&redirect_uri='+BASE_URL
                }]
            }
        }
    return json.loads(json.dumps(response, default=str))
```

And that's it. Now you can implement your authorization pipeline using Python, Lambda@Edge, CloudFront and S3.
