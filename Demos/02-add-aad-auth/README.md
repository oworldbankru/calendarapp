# Completed module: Add Azure AD authentication

The version of the project in this directory reflects completing the tutorial up through [Add Azure AD authentication](https://docs.microsoft.com/graph/training/node-tutorial?tutorial-step=3). If you use this version of the project, you need to complete the rest of the tutorial starting at [Get calendar data](https://docs.microsoft.com/graph/training/node-tutorial?tutorial-step=4).

> **Note:** It is assumed that you have already registered an application in the app registration portal as specified in [Register the app in the portal]. You need to configure this version of the sample as follows:
>
In this exercise you will extend the application from the previous exercise to support authentication with Azure AD. This is required to obtain the necessary OAuth access token to call the Microsoft Graph. In this step you will integrate the passport-azure-ad library into the application.

Create a new file named .env file in the root of your application, and add the following code.

```
OAUTH_APP_ID=YOUR_APP_ID_HERE
OAUTH_APP_PASSWORD=YOUR_APP_PASSWORD_HERE
OAUTH_REDIRECT_URI=http://localhost:3000/auth/callback
OAUTH_SCOPES='profile offline_access user.read calendars.read'
OAUTH_AUTHORITY=https://login.microsoftonline.com/common/
OAUTH_ID_METADATA=v2.0/.well-known/openid-configuration
OAUTH_AUTHORIZE_ENDPOINT=oauth2/v2.0/authorize
OAUTH_TOKEN_ENDPOINT=oauth2/v2.0/token
```
Replace YOUR APP ID HERE with the application ID from the Application Registration Portal, and replace YOUR APP SECRET HERE with the password you generated.

Open ./app.js and add the following line to the top of the file to load the .env file.

```
require('dotenv').config();
```

Implement sign-in

Locate the line var indexRouter = require('./routes/index'); in ./app.js. Insert the following code before that line.

```
var passport = require('passport');
var OIDCStrategy = require('passport-azure-ad').OIDCStrategy;

// Configure passport

// In-memory storage of logged-in users
// For demo purposes only, production apps should store
// this in a reliable storage
var users = {};

// Passport calls serializeUser and deserializeUser to
// manage users
passport.serializeUser(function(user, done) {
  // Use the OID property of the user as a key
  users[user.profile.oid] = user;
  done (null, user.profile.oid);
});

passport.deserializeUser(function(id, done) {
  done(null, users[id]);
});

// Callback function called once the sign-in is complete
// and an access token has been obtained
async function signInComplete(iss, sub, profile, accessToken, refreshToken, params, done) {
  if (!profile.oid) {
    return done(new Error("No OID found in user profile."), null);
  }

  // Save the profile and tokens in user storage
  users[profile.oid] = { profile, accessToken };
  return done(null, users[profile.oid]);
}

// Configure OIDC strategy
passport.use(new OIDCStrategy(
  {
    identityMetadata: `${process.env.OAUTH_AUTHORITY}${process.env.OAUTH_ID_METADATA}`,
    clientID: process.env.OAUTH_APP_ID,
    responseType: 'code id_token',
    responseMode: 'form_post',
    redirectUrl: process.env.OAUTH_REDIRECT_URI,
    allowHttpForRedirectUrl: true,
    clientSecret: process.env.OAUTH_APP_PASSWORD,
    validateIssuer: false,
    passReqToCallback: false,
    scope: process.env.OAUTH_SCOPES.split(' ')
  },
  signInComplete
));
```
This code initializes the Passport.js library to use the passport-azure-ad library, and configures it with the app ID and password for the app.

Now pass the passport object to the Express app. Locate the line app.use('/', indexRouter); in ./app.js. Insert the following code before that line.

```
// Initialize passport
app.use(passport.initialize());
app.use(passport.session());
```
Create a new file in the ./routes directory named auth.js and add the following code.

```
var express = require('express');
var passport = require('passport');
var router = express.Router();

/* GET auth callback. */
router.get('/signin',
  function  (req, res, next) {
    passport.authenticate('azuread-openidconnect',
      {
        response: res,
        prompt: 'login',
        failureRedirect: '/',
        failureFlash: true,
        successRedirect: '/'
      }
    )(req,res,next);
  }
);

router.post('/callback',
  function(req, res, next) {
    passport.authenticate('azuread-openidconnect',
      {
        response: res,
        failureRedirect: '/',
        failureFlash: true
      }
    )(req,res,next);
  },
  function(req, res) {
    // TEMPORARY!
    // Flash the access token for testing purposes
    req.flash('error_msg', {message: 'Access token', debug: req.user.accessToken});
    res.redirect('/');
  }
);

router.get('/signout',
  function(req, res) {
    req.session.destroy(function(err) {
      req.logout();
      res.redirect('/');
    });
  }
);

module.exports = router;
```

This defines a router with three routes: signin, callback, and signout.

The signin route calls the passport.authenticate method, causing the app to redirect to the Azure login page.

The callback route is where Azure redirects after the signin is complete. The code calls the passport.authenticate method again, causing the passport-azure-ad strategy to request an access token. Once the token is obtained, the next handler is called, which redirects back to the home page with the access token in the temporary error value. We'll use this to verify that our sign-in is working before moving on. Before we test, we need to configure the Express app to use the new router from ./routes/auth.js.

The signout method logs the user out and destroys the session.

Insert the following code before the var app = express(); line.
```
var authRouter = require('./routes/auth');
```

Then insert the following code after the app.use('/', indexRouter); line.
```
app.use('/auth', authRouter);
```
Start the server and browse to https://localhost:3000. Click the sign-in button and you should be redirected to https://login.microsoftonline.com. Login with your Microsoft account and consent to the requested permissions. The browser redirects to the app, showing the token.
Get user details

Start by creating a new file to hold all of your Microsoft Graph calls. Create a new file in the root of the project named graph.js and add the following code.

```
var graph = require('@microsoft/microsoft-graph-client');
require('isomorphic-fetch');

module.exports = {
  getUserDetails: async function(accessToken) {
    const client = getAuthenticatedClient(accessToken);

    const user = await client.api('/me').get();
    return user;
  }
};

function getAuthenticatedClient(accessToken) {
  // Initialize Graph client
  const client = graph.Client.init({
    // Use the provided access token to authenticate
    // requests
    authProvider: (done) => {
      done(null, accessToken);
    }
  });

  return client;
}
```

This exports the getUserDetails function, which uses the Microsoft Graph SDK to call the /me endpoint and return the result.

Update the signInComplete method in /app.js to call this function. First, add the following require statements to the top of the file.

```
var graph = require('./graph');
```

Replace the existing signInComplete function with the following code.

```
async function signInComplete(iss, sub, profile, accessToken, refreshToken, params, done) {
  if (!profile.oid) {
    return done(new Error("No OID found in user profile."), null);
  }

  try{
    const user = await graph.getUserDetails(accessToken);

    if (user) {
      // Add properties to profile
      profile['email'] = user.mail ? user.mail : user.userPrincipalName;
    }
  } catch (err) {
    done(err, null);
  }

  // Save the profile and tokens in user storage
  users[profile.oid] = { profile, accessToken };
  return done(null, users[profile.oid]);
}
```
The new code updates the profile provided by Passport to add an email property, using the data returned by Microsoft Graph.

Finally, add code to ./app.js to load the user profile into the locals property of the response. This will make it available to all of the views in the app.

Add the following after the app.use(passport.session()); line.

```
app.use(function(req, res, next) {
  // Set the authenticated user in the
  // template locals
  if (req.user) {
    res.locals.user = req.user.profile;
  }
  next();
});
```

Storing the tokens

Now that you can get tokens, it's time to implement a way to store them in the app. Currently, the app is storing the raw access token in the in-memory user storage. Since this is a sample app, for simplicity's sake, you'll continue to store them there. A real-world app would use a more reliable secure storage solution, like a database.

However, storing just the access token doesn't allow you to check expiration or refresh the token. In order to enable that, update the sample to wrap the tokens in an AccessToken object from the simple-oauth2 library.

First, in ./app.js, add the following code before the signInComplete function.

```
// Configure simple-oauth2
const oauth2 = require('simple-oauth2').create({
  client: {
    id: process.env.OAUTH_APP_ID,
    secret: process.env.OAUTH_APP_PASSWORD
  },
  auth: {
    tokenHost: process.env.OAUTH_AUTHORITY,
    authorizePath: process.env.OAUTH_AUTHORIZE_ENDPOINT,
    tokenPath: process.env.OAUTH_TOKEN_ENDPOINT
  }
});
```

Then, update the signInComplete function to create an AccessToken from the raw tokens passed in and store that in the user storage. Replace the existing signInComplete function with the following.

```
async function signInComplete(iss, sub, profile, accessToken, refreshToken, params, done) {
  if (!profile.oid) {
    return done(new Error("No OID found in user profile."), null);
  }

  try{
    const user = await graph.getUserDetails(accessToken);

    if (user) {
      // Add properties to profile
      profile['email'] = user.mail ? user.mail : user.userPrincipalName;
    }
  } catch (err) {
    done(err, null);
  }

  // Create a simple-oauth2 token from raw tokens
  let oauthToken = oauth2.accessToken.create(params);

  // Save the profile and tokens in user storage
  users[profile.oid] = { profile, oauthToken };
  return done(null, users[profile.oid]);
}
```

Update the callback route in ./routes/auth.js to remove the req.flash and manual redirect, and provide the successRedirect parameter to passport.authenticate. The callback route should look like the following.

```
router.post('/callback',
  function(req, res, next) {
    passport.authenticate('azuread-openidconnect',
      {
        response: res,
        failureRedirect: '/',
        failureFlash: true,
        successRedirect: '/'
      }
    )(req,res,next);
  }
);
```

Restart the server and go through the sign-in process. You should end up back on the home page, but the UI should change to indicate that you are signed-in.
