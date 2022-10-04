# Node.js - Demo Web Application

This is a simple Node.js web app using the Express framework and EJS templates.

> NOTE: This repository is a copy of the https://github.com/benc-uk/nodejs-demoapp, created by Ben Coleman (@benc-uk). It has been re-structured and tweaked to facilitate deploying this application in Tanzu Application Platform.

Table of Contents
- [Application Description](#application-description)
- [Run in a Docker Container](#run-in-a-docker-container)
- [Deploy on Kubernetes](#deploy-on-kubernetes)
- [Deploy on Tanzu Application Platform](#deploy-on-tanzu-application-platform)
- [Optional Features](#optional-features)
    - [Application Insights](#application-insights)
    - [Weather Details](#weather-details)
    - [User Authentication with Azure AD](#user-authentication-with-azure-ad)
    - [Todo App](#todo-app)
- [Configuration](#configuration)

## Application Description

The app has been designed with cloud native demos & containers in mind, in order to provide a real working application for deployment, something more than "hello-world" but with the minimum of pre-reqs. It is not intended as a complete example of a fully functioning architecture or complex software design.

Typical uses would be deployment to Kubernetes, demos of Docker, CI/CD (build pipelines are provided), deployment to cloud (Azure) monitoring, auto-scaling

The app has several basic pages accessed from the top navigation menu, some of which are only lit up when certain configuration variables are set (see 'Optional Features' below):

- **'Info'** - Will show system & runtime information, and will also display if the app is running from within a Docker container and Kubernetes.
- **'Tools'** - Some tools useful in demos, such a forcing CPU load (for autoscale demos), and error/exception pages for use with App Insights or other monitoring tool.
- **'Monitor'** - Display realtime monitoring data, showing memory usage/total and process CPU load.
- **'Weather'** - (Optional) Gets the location of the client page (with HTML5 Geolocation). The resulting location is used to fetch weather data from the [OpenWeather](https://openweathermap.org/) API
- **'Todo'** - (Optional) This is a small todo/task-list app which uses MongoDB as a database.
- **'User Account'** - (Optional) When configured with Azure AD (application client id) user login button will be enabled, and an user-account details page enabled, which calls the Microsoft Graph API

![screen](https://user-images.githubusercontent.com/14982936/55620043-dfe96480-5791-11e9-9746-3b42a3a41e5f.png)
![screen](https://user-images.githubusercontent.com/14982936/55620045-dfe96480-5791-11e9-94f3-6d788ed447c1.png)
![screen](https://user-images.githubusercontent.com/14982936/58764072-d8102b80-855a-11e9-993f-21ef0344d5e0.png)

## Run in a Docker Container

Public container image is [available on GitHub Container Registry](https://github.com/users/benc-uk/packages/container/package/nodejs-demoapp).

Run in a container with:

```bash
docker run --rm -it -p 3000:3000 ghcr.io/benc-uk/nodejs-demoapp:latest
```

Should you want to build your own container, run `make image` from the `other` folder and the above variables to customise the name & tag.

## Deploy on Kubernetes

The app can easily be deployed to Kubernetes using Helm, see [other/deploy/kubernetes/readme.md](other/deploy/kubernetes/readme.md) for details

## Deploy on Tanzu Application Platform

If you are going to use a `supply-chain` with tests you will require to deploy this Tekton Pipeline in advance: `/config/tekton-pipeline-node.yaml`. Notice we force installation of the `prom-client` package hat might not be available in other Pipelines.
```shell script
kubectl apply -f ./config/tekton-pipeline-node.yaml
```

Use the provided `/config/workload.yaml` to deploy the application into TAP
```shell script
tanzu apps workload create nodejs-demoapp \
  -f ./config/workload.yaml \
  --yes
```

Use the provided `/config/catalog-info.yaml` to import the application into the TAP-GUI

## Optional Features

The app will start up and run with zero configuration, however the only features that will be available will be the INFO and TOOLS views. The following optional features can be enabled:

### Application Insights

Enable this by setting `APPLICATIONINSIGHTS_CONNECTION_STRING`

The app has been instrumented with the Application Insights SDK, it will however need to be configured to point to your App Insights instance/workspace. All requests will be tracked, as well as dependant calls to MongoDB or other APIs (if configured), exceptions & error will also be logged

[This article](https://docs.microsoft.com/azure/application-insights/app-insights-nodejs) has more information on monitoring Node.js with App Insights

### Weather Details

Enable this by setting `WEATHER_API_KEY`

This will require a API key from OpenWeather, you can [sign up for free and get one here](https://openweathermap.org/price). The page uses a browser API for geolocation to fetch the user's location.  
However, the `geolocation.getCurrentPosition()` browser API will only work when the site is served via HTTPS or from localhost. As a fallback, weather for London, UK will be show if the current position can not be obtained

### User Authentication with Azure AD

Enable this by setting `AAD_APP_ID`

This uses [Microsoft Authentication Library (MSAL) for Node](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-node) to authenticate via MSAL with OIDC and OAuth 2.0. The flow it uses is the "Authorization Code Grant (PKCE)", which means we can sign in users without needing client secrets

In addition the user account page shows details & photo retrieved from the Microsoft Graph API

You will need to register an app in your Azure AD tenant. The app should be configured for the PKCE flow, if creating the app via the portal select **_Public client/native (mobile & desktop)_** (ignore the fact this doesn't seem the right option for a web app)

When configuring authentication the redirect URL will be the host where the app is running with `/signin` as the URL path, e.g. `https://myapp.azurewebsites.net/signin`, for local testing use `http://localhost:3000/signin`

For the signin audience select **_Accounts in any organizational directory (Any Azure AD directory - Multitenant) and personal Microsoft accounts (e.g. Skype, Xbox)_**

To simplify the registration, the Azure CLI can be used with the following bash snippet:

```bash
baseUrl="http://localhost:3000"
name="NodeJS Demo"
# Create app registration and get client ID
clientId=$(az ad app create \
--public-client-redirect-uris "$baseUrl/signin" \
--display-name "$name" \
--sign-in-audience AzureADandPersonalMicrosoftAccount \
--query appId -o tsv)
# Create a service principal for the application
az ad sp create --id $clientId -o json
echo -e "\n### Set env var AAD_APP_ID to '$clientId'"
```

### Todo App

Enable this by setting `TODO_MONGO_CONNSTR`

A mini todo & task tracking app can be enabled if a MongoDB backend is provided and a connection string to access it. This feature is primarily to show database dependency detection and tracking in App Insights

The default database name is `todoDb` but you can change this by setting `TODO_MONGO_DB`

You can stand up MongoDB in a container instance or in Cosmos DB (using the Mongo API). Note. When using Cosmos DB and the _per database provisioned RU/s_ option, you must manually create the collection called `todos` in the relevant database and set the shard key to `_id`

## Configuration

The following configuration environmental variables are supported, however none are mandatory. These can be set directly or when running locally will be picked up from an `.env` file if it is present. A sample `.env` file called `.env.sample` is provided for you to copy

If running in an Azure Web App, all of these values can be injected as application settings in Azure.

| Environmental Variable                | Default | Description                                                                      |
| ------------------------------------- | ------- | -------------------------------------------------------------------------------- |
| PORT                                  | 3000    | Port the server will listen on                                                   |
| TODO_MONGO_CONNSTR                    | _none_  | Connect to specified MongoDB instance, when set the todo feature will be enabled |
| TODO_MONGO_DB                         | todoDb  | Name of the database in MongoDB to use (optional)                                |
| APPLICATIONINSIGHTS_CONNECTION_STRING | _none_  | Enable Application Insights monitoring                                           |
| WEATHER_API_KEY                       | _none_  | OpenWeather API key. [Info here](https://openweathermap.org/api)                 |
| AAD_APP_ID                            | _none_  | Client ID of app registered in Azure AD                                          |
| DISABLE_METRICS                       | _none_  | Set to truthy value if you want to switch off Prometheus metrics                 |
| REDIS_SESSION_HOST                    | _none_  | Point to a Redis host to hold/persist session cache                              |
