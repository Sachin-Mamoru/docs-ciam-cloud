# Try organization login

Let's try organization login using a sample scenario.

## Scenario

**Guardio** is an auto insurance service company that has many employees who use different credentials to sign in to different internal organizational applications. **Guardio-SaaS-App** is one such applicaiton. Here, Guardio Insurance is using the WSO2 Identity Server B2B Private CIAM Solution, therefore it’s configured as the super organization.

<img src="../../../assets/img/guides/organization-login/try-it-out/scenario_diagram.png" alt="Scenario Diagram" width="700">

While Larry is an administrator at Guardio Insuarnce super organization, following are the suborganizations and the respective admins.

-   **Best Auto Mart**: Alex
-   **Car Traders**: Sam

Larry wants to register a user account in each of the suborganizations and provide an organization specific login option for the users to log in to the respective suborganization IDP.

## Prerequisites

Create [Organization SSO Login](../configure-organization-idp) in the root business (the super organization) as a federated authenticator.

## Step 1: Create the organizations

1.  Sign in to the super organization.
    -   Sign in to the super organization on the console `https://<SERVER_HOST>:9443/console`.
    -   Sign in to another organization on the console `https://<SERVER_HOST>:9443/o/<organization id>/console`.
2.  Create the following suborganizations:

    -   Best Auto Mart
    -   Car Traders

    <img src="../../../assets/img/guides/organization-login/try-it-out/created_sub_organizations.png" alt="Created suborganizations" width="700" style="border:1px solid grey">

## Step 2: Configure the business apps

1.  Configuring Guardio SaaS application
2.  Create the OIDC SaaS application in the super organization.

    !!! note
        Be sure to select **Management Application** when creating the application.
        
        <img src="../../../assets/img/guides/organization-login/create-business-applications/management_application.png" alt="Management Application" width="300" style="border:1px solid grey">

3.  Replace the consumerKey and consumerSecret values with the OAuth client key and OAuth client secret values that were generated for the newly created application.

    !!! note
        Be sure to select the [Organization Switch](../../../references/org-domains-urls) and Code grant grant types in the application.

    <img src="../../../assets/img/guides/organization-login/try-it-out/app_oidc_config.png" alt="App OIDC Configurations" width="700" style="border:1px solid grey">

4.  In the created SaaS Application, under Sign-in Method add Organization SSO Login for authentication. Other than that add the following adaptive script for conditional authentication under organization idp.

    !!! note
        Remove the **username & password** authentication step from the sign-in flow of the application. By doing so, you will be directly to the selected organization IdP for authentication.

    <img src="../../../assets/img/guides/organization-login/try-it-out/saas_app_sign_in_with_sample_app.png" alt="Saas App Sign-in with Sample App" width="700" style="border:1px solid grey">

    ``` js
    var onLoginRequest = function(context) {
    executeStep(1,
        {
            authenticationOptions: [{
                idp: (context.request.params.org && !context.steps[1].idp) ? "<ORG IDP NAME>" : context.steps[1].idp
            }],
            authenticatorParams : {
                common : {
                    'skipIdentifierPreProcess' : "true"
                }
            }
        },
        {
            onSuccess: function (context) {
                Log.info("User successfully completed initial with IDP : " + context.steps[1].idp);
                if (context.steps[1].idp === "<ORG IDP NAME>") {
                    return;
                }
            }
        });
    };
    ```

## Step 3: Share the business app with suborganizations

<img src="../../../assets/img/guides/organization-login/try-it-out/share_app_with_sub_orgs.png" alt="Share App with suborganizations" width="400">

For the user base of Best Auto Mart organization, create a separate [Asgardeo IdP](https://wso2.com/asgardeo/docs/guides/authentication/oidc/). 

## Step 4: Onboard suborganization IdPs

Create an application in [Asgardeo](https://asgardeo.io/signup) with the OAuth2.0/OpenID Connect protocol and get the client id and secret to create an IdP for the Best Auto Mart organization. Also, create Alex as the admin.

!!! note
    Create a user with the required [permissions](../../b2b-org-management/b2b-org-permissions) to create an IdP in the suborganization.
  
<img src="../../../assets/img/guides/organization-login/try-it-out/asgardeo_app_info.png" alt="Asgardeo App Info" width="700" style="border:1px solid grey">

Use the above information to create the OIDC-based Asgardeo IdP.

In the fragmented SaaS Application of the Best Auto Mart organization, add the created Asgardeo IdP for authentication in the sign-in flow.
  
<img src="../../../assets/img/guides/organization-login/try-it-out/asgardeo_idp_in_fragment_app.png" alt="Asgardeo IdP in Fragment App" width="700" style="border:1px solid grey">

For the Car Traders organization, [create the user](../../org-user-management) Sam with admin [permissions](../../b2b-org-management/b2b-org-permissions) in the user base.

## Try it out

### Step 1: Setting up the sample application

1.  Download the [sample application](https://github.com/Achintha444/guardio-insurance-sample-application/archive/main.zip).
2.  Extract the content of the zip file.
3.  Open the `config.json` file found in the `<SAMPLE_APP_HOME>/guardio-insurance-sample-application-main` directory and configure the following properties:

    ``` json
    "WSO2IS_HOST": <Identity Server URL>,
    "WSO2IS_CLIENT_ID": <APPLICATION CLIENT ID>, 
    "WSO2IS_CLIENT_SECRET": <APPLICATION CLIENT SECRET>,
    "SAMPLE_ORGS" : [
        {
            "id" : <Best Auto Mart Organization ID>,
            ...
        },
        {
            "id" : <Car Traders Organization ID>,
            ...
        },
    ]
    ```

### Step 2: Setting up the WSO2 Identity Server

1.  Add the following configurations to the `<IS_HOME>/repository/conf/deployment.toml` file to allow HTTP POST requests:

    ``` toml
    allow_generic_http_requests = true
    allow_any_origin = false
    allowed_origins = [
    "http://localhost:3000"
    ]
    allow_subdomains = false
    supported_methods = [
        "GET",
        "POST",
        "HEAD",
        "OPTIONS",
        "PUT",
        "DELETE"	
    ]
    support_any_header = true
    supported_headers = []
    exposed_headers = []
    supports_credentials = true
    max_age = 3600
    tag_requests = false
    ```

2.  On the Management Console, go to the **Protocol** tab and add the following configurations:

    -   Add the below callback URL as Authorized redirect URL to the application.
        `http://localhost:3000/api/auth/callback/wso2is`
    -   Add the below URL as the allowed origin.
        `http://localhost:3000`

### Step 3: Start the sample application

1.  Open a terminal, navigate to  the `<SAMPLE_APP_HOME>/guardio-insurance-sample-application-main` folder and run the following commands respectively to start the application:

    !!! info
        To complete this step, it is required to have node installed in your machine.

    ``` node
    npm install
    npm run dev
    ```

2.  To check the sample application, go to `http://localhost:3000` on your browser.

3.  Click **Sign In** to proceed in the application.

    <img src="../../../assets/img/guides/organization-login/try-it-out/sample_app_sign_in.png" alt="Sample App Sign-in" width="700" style="border:1px solid grey">

4.  Select an organization from the list and click **Next**. In this example, we will select Car Traders.

    <img src="../../../assets/img/guides/organization-login/try-it-out/sample_app_org.png" alt="Sample App Orgs" width="700" style="border:1px solid grey">

5.  Enter the user credentials to log in to the Car Traders Organization.

    <img src="../../../assets/img/guides/organization-login/try-it-out/sign_in_car_traders_org.png" alt="Sign-in Car Traders Organization" width="700" style="border:1px solid grey">

6.  Now, you have successfully logged in to the sample application.

    <img src="../../../assets/img/guides/organization-login/try-it-out/logged_in_to_sample_app.png" alt="Logged in to Sample App" width="700" style="border:1px solid grey">

7.  Now, select Best Auto Mart Organization and try to log in. You’ll be redirected to the Asgardeo IdP based login page.

    <img src="../../../assets/img/guides/organization-login/try-it-out/asgardeo_based_login.png" alt="Asgardeo based Login" width="400" style="border:1px solid grey">
