---
layout: post
title: Upload files as multipart/form-data from python flask server to third party API using requests
date: 2022-10-10 11:00:00 -0500
description: Understand how to upload files as multipart/form-data from flask sever to external API using requests module
img: posts_imgs/requests-toolbelt-multipart-formdata/python_requests_toolbelt.jpeg
tags: [python, flask, requests,  API, server, backend, form-data, MultipartEncoder]
---

Recently, i was developing a python module to integrate electronic billing in Mexico to my startup. After doing some research, i found [facturapi](https://www.facturapi.io/) which offer a very easy to implement and high escalable API to generate electronic bills. In Mexico, in order to generate an electronic bill approved by the local tax authority it is required to "sign" the bills with a digital certificate and key so the tax authority can validate the bill.

So, to enable our cusotmers to generate electronic bills, they had to upload the certificate, the key and the certificate password to facturapi as part of the initial setup. Since the integration was going to work to multiple organizations, it was necessary to include in the code the procedure to upload the information from our app and send it to facturapi.

The integration module was developed in Python, however at the moment of writing this post, facturapi did not offer SDK/Documentation for python integration (Only cURL, PHP, Node.js and C#). Using requests library from python it is easy to generate the REST resources following cURL documentation.

Facturapi documentation to upload the files can be found [here](https://docs.facturapi.io/api/#tag/organization/operation/uploadOrganizationCertificate)

In the frontend side, the form to upload the files and fill the password is shown below:

```html
    <div class="form-row">
        <div class="custom-file col-md-4">
            <input class="custom-file-input" id="cerFile" name="cerFile" required="" type="file">
            <label class="custom-file-label" for="cerFile" data-browse="Elegir">Archivo .cer</label>
        </div>
        <div class="custom-file col-md-4">
            <input class="custom-file-input" id="keyFile" name="keyFile" required="" type="file">
            <label class="custom-file-label" for="keyFile" data-browse="Elegir">Archivo .key</label>
        </div>
        <div class="form-group col-md-4">
            <input class="form-control" id="pwd_cer" name="cert_pwd" placeholder="Contraseña del certificado" required="" type="password" value="">
        </div>
    </div>
```

It is very important that the FORM tag include the type of data to be sent through the post request as multipart/form-data.

```html
    <form method="POST" action="/path-to-process-post-method" enctype="multipart/form-data">
```

Set enctype in the form tag as "multipart/form-data" is required because this is how facturaapi needs to receive the data sent. This form points to our server, and we need to process this request and perform another request with the same information (suggest to validate everyhing before process) from our server to facturapi.

In our backend, after validating that the data received in the form is correct, we generate the request to upload the certificates and the password to facturapi using a "PUT" request.

But first we need to install some dependencies required to perform requests from our server to a third party API.

```console
{
    pip3 install requests
    pip3 install requests-toolbelt
}
```

Backend code to upload the certificates is inside below function.

```python
    import requests
    from requests_toolbelt.multipart.encoder import MultipartEncoder

    def upload_csd(organizationId, cerFile, keyFile, cerPwd):
        '''
            Upload CSD of the organization to facturapi
        '''
        facturapi_user_key = app.config['MX_FACTURAPI_USER_KEY']
        url = 'https://www.facturapi.io/v2/organizations/'+organizationId+'/certificate'
        payload = MultipartEncoder(
                    fields={
                        'password':cerPwd,
                        'cer': (cerFile.filename, cerFile.read(), 'application/octet-stream'),
                        'key': (keyFile.filename, keyFile.read(), 'application/octet-stream')
                    }
                )
        headers = {
            'Content-Type': payload.content_type,
            'Authorization': str('Bearer ' + facturapi_user_key)
        }
        response = requests.request("PUT", url, headers=headers, data=payload)
        return response
```

Now, lets review the code use to send data to facturapi.

### Import dependencies

Libraries imported are required to perfom requests from python. The "Requests" module is very common used in python servers so there is no need to explain more about it, it basically allows us to perform HTTP requests (GET, POST, PUT, DELETE etc.)

The important part here is the "Requests_toolbelt" module.

```python
    from requests_toolbelt.multipart.encoder import MultipartEncoder
```

This is very important to consider when sending "multipart/form-data" because "Requests" library alone dont handle pretty well this type of form-data and specially for this case, uploading the files and password will not work unless we use this library along with Requests.

### Information required to upload certificate, key and password to facturapi

According to the facturapi documentation, there are several parameters we need to consider in our HTTP request.

1. URL Endpoint
    For this specific request, the endpoint url is "https://www.facturapi.io/v2/organizations/<organizationId>/certificate'

2. Organization Id
    The organizationId is generated when a new organization is registered in facturapi, if you have several clients to offer electronic billing, you should generate 1 organization for each client.
    The organization id is used in the URL endpoint for this request.

3. Facturapi user key
    This is a string generated per facturapi account. You can get this key when loging into facturapi dasboard. This key is used to manage all organizations.
    In this request, the facturapi user key is sent on the headers using following key/value pair
    ```python
        'Authorization': str('Bearer ' + facturapi_user_key)
    ```

### Generate payload data to include in the request

The payload to be sent in the request can be found in the following code:

```python
    payload = MultipartEncoder(
                    fields={
                        'password':cerPwd,
                        'cer': (cerFile.filename, cerFile.read(), 'application/octet-stream'),
                        'key': (keyFile.filename, keyFile.read(), 'application/octet-stream')
                    }
                )
```

The data to be sent are:

1. Password
    It is sent as string (how it is received from the frontend)

2. Cer and Key
    The files must be send in binary stream, so for each we send a tuple with the filename, binary stream data and content type.

### MultipartEncoder to send multipart/form-data with rqeuests

As we can see in previous code snippet, to generate the payload we use the method "MultipartEncoder" which is from the library "Requests_toolbelt".

The reason why we need to consider this funciton to generate the payload to be sent is because we need to send 'Boundary' key/pair in the header. If this is not included, then the facturapi server will receive the full data and it wont be able to separate each field (password, cer, key). The 'Boundary' is a string that allows facturapi server to know where is the limit for each field, so it can get 3 separate fields as required.

The Requests module [shows](https://requests.readthedocs.io/en/latest/user/quickstart/#post-a-multipart-encoded-file) in its documentation that for Multipart data it might be necessary to use the toolbelt which is an additional module not included in Requests library.

[Here](https://stackoverflow.com/questions/3508338/what-is-the-boundary-in-multipart-form-data) you can find some information about the meaning of Boundary


And that's it, I hope this post is useful when you need to implement multipart/form-data along with requests module to send data from your website to an external API through your backend server.