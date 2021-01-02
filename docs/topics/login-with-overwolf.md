---
id: login-with-overwolf
title: App login with Overwolf
sidebar_label: Login with Overwolf
---

This article will explain how to implement an Overwolf login/auth interface in your Overwolf app or your Website. This flow is web browser flow only, not API-based authentication.

## Login flow overview

The login form is hosted on the OW servers. That means you should implement only a "Login with Overwolf" button on your website/app. You can't directly login from your website/app, as we currently do not offer client SDK that supports API-based authentication.

The login button should open the OW login page (https://accounts.overwolf.com/oauth2/), hosted on the OW servers, in a new browser tab or popup window.  

Once the login is done, the user is redirected to a pre-defined redirect_URI hosted on your servers.

Note that we support the OW URI scheme (overwolf-extension://..), so if you implement the login in your OW app, the redirect_URI should be the name of one of your app's windows. 

## Prerequisite

### Register your app on Overwolf

To implement OW login in your app or website, you first need to register your app on Overwolf (send us an email to developers@overwolf.com) to generate your `client_id` and `client_secret`.

You should provide these parameters:

1. client_name - The app's name.
2. redirect_uris - List of callback URL's (at least one).  
   An endpoint on your server/app. More details [here](#create-redirect_uri-endpoint).  
3. logo_uri - URL of the app's logo.
4. policy_uri - URL of the app's policy.
5. tos_uri - URL of the app's "Terms of Service".

Once the registration is completed, you will get from us your app's `client_id` and your `client_secret`.

### Create redirect_uri endpoint

On your server, create a URL that will be used to get the auth-token from Overwolf.  
[Here](#3-implement-your-app-s-server-side-endpoint) you can find more details on the required server side code for this page.

If you implement the OW login in your OW app, you can use existing app window, or create a dedicate window for that. 

### Implement your app's server side endpoint.

On your server, create a URL that will be used to get the auth-token from Overwolf, by doing a POST request to OW server.

In our example, this page is http://localhost:5000/oidc-callback. 

In the code behind of this page, you should do a POST request with the following details:

```js
https://accounts.overwolf.com/oauth2/token?client_id={client id}&client_secret={client secret}&grant_type=authorization_code&code={code that came from request object, e.g: request.query.code}&redirect_uri={redirect_uri}
```

#### Requiree Query params:

* client_id
* client_secret
* grant_type
* code 
* redirect_uri

#### Required HTTP headers:

* Content-Type: "application/x-www-form-urlencoded"

At this point, you should get a response from the POST request, with the auth token (or error). More info on [step 3](3-get-the-auth-token).

## 1. Engage the SSO flow.

The first step is to implemet a login button in your website/app.  
The button click should open a new tab or popup window that redirect to the OW login page, by implementing this GET request:

```js
GET https://accounts.overwolf.com/oauth2/auth?response_type=code&client_id={client id}&redirect_uri={redirect_uri}&scope={desired scope separated by '+', e.g: openid+profile+email}
```

#### Requiree Query params:

* response_type
* client_id
* redirect_uri
* scope

#### Sample code that implemeting the above

This is an example of a server side code that runs once the user clicks on the "Login with Overwolf" button. 

As you can see, the `redirect_uri` is `http://localhost:5000/oidc-callback`. This is page that recive the auth token (or login error) once the auth process is finished on the OW side.

```js
var express = require('express');
var router = express.Router();
const axios = require('axios');
const querystring = require('querystring');

const demoClient = {
  client_id: 'xxxxxxxx',
  client_secret: 'yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy',
};

/**
 * start the auth flow by opening this in the browser:
 * https://accounts.overwolf.com/oauth2/auth?response_type=code&client_id=14p5m4qp90bya1svhql1r04k0rk1yn80&redirect_uri=http://localhost:5000/oidc-callback&scope=openid+profile+email
 *
 * this is the callback endpoint as passed in the redirect_uri parameter
 * and should be whitelisted in the oauth client application
 */
router.get('/oidc-callback', function(req, res, next) {
  const client = demoClient;
  axios.post('https://accounts.overwolf.com/oauth2/token',
    querystring.stringify({
      client_id: client.client_id,
      client_secret: client.client_secret,
      grant_type: 'authorization_code',
      code: req.query.code,
      redirect_uri: 'http://localhost:5000/oidc-callback'
    }),
    {
      headers: {
        "Content-Type": "application/x-www-form-urlencoded"
      }
    }).then(function(response) {
      // the response will contain the access token to be used
      console.log(response);
      res.json({});
    }).catch((e) => {
      console.error(e)
      res.send('err')
  });
});

router.get('/oidcresult', function(req, res, next) {
  res.json(req.query);
});

module.exports = router;
```

## 2. Login on Overwolf

Once the SSO flow is engaged, the user is redirected to the OW hosted login page:

![OW login screenshot](assets/ow_login.png)

On succesful login, the user gets also a consent screen to requested scopes.  
After the consent, we will redirect the user back to the redirect_uri (on the above example is http://localhost:5000/oidc-callback).



## 3. Get the auth token

If there was no error in the flow, you should get a response from the POST request with the following auth token details:

* access_token
* expires_in
* id_token
* scope
* token_type

You can save the auth token in a localStorage variable. We recommend encrypting the hash/token before storing it - for security reasons. You can use the [overwolf.cryptography API](../api/overwolf-cryptography) for that.

## 4. Close the Log In window.

Now, we can safely close the login window.

The login process is complete.