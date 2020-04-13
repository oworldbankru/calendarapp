# Completed module: Create a Node.js Express web app

The version of the project in this directory reflects completing the tutorial up through [Create a Node.js Express web app](https://docs.microsoft.com/graph/training/node-tutorial?tutorial-step=1). If you use this version of the project, you need to complete the rest of the tutorial starting at [Register the app in the portal](https://docs.microsoft.com/graph/training/node-tutorial?tutorial-step=2).

Create a Node.js Express web app

    30 minutes remaining

In this exercise you will use Express to build a web app. If you don't already have the Express generator installed, you can install it from your command-line interface (CLI) with the following command.


    npm install express-generator -g

Open your CLI, navigate to a directory where you have rights to create files, and run the following command to create a new Express app that uses Handlebars as the rendering engine.


    express --hbs graph-tutorial

The Express generator creates a new directory called graph-tutorial and scaffolds an Express app. Navigate to this new directory and enter the following command to install dependencies.


    npm install

Once that command completes, use the following command to start a local web server.


    npm start

Open your browser and navigate to http://localhost:3000. If everything is working, you will see a "Welcome to Express" message. If you don't see that message, check the Express getting started guide.

Before moving on, install some additional gems that you will use later:

    dotenv for loading values from a .env file.
    moment for formatting date/time values.
    connect-flash to flash error messages in the app.
    express-session to store values in an in-memory server-side session.
    passport-azure-ad for authenticating and getting access tokens.
    simple-oauth2 for token management.
    microsoft-graph-client for making calls to Microsoft Graph.
    isomorphic-fetch to polyfill the fetch for Node. A fetch polyfill is required for the microsoft-graph-client library. See the Microsoft Graph JavaScript client library wiki for more information.

Run the following command in your CLI.


    npm install dotenv@8.2.0 moment@2.24.0 connect-flash@0.1.1 express-session@1.17.0 isomorphic-fetch@2.2.1
    npm install passport-azure-ad@4.2.0 simple-oauth2@3.1.0 @microsoft/microsoft-graph-client@2.0.0



Now update the application to use the connect-flash and express-session middleware. Open the ./app.js file and add the following require statement to the top of the file.
JavaScript

    var session = require('express-session');
    var flash = require('connect-flash');
