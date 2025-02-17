# Run the Surveys application

This page describes how to run the [Tailspin Surveys](./README.md) application locally, from Visual Studio. In these steps, you won't deploy the application to Azure. However, you will need to create some Azure resources &mdash; an Azure Active Directory (Azure AD) directory and a Redis cache.

Here is a summary of the steps:

1. Create an Azure AD directory (tenant) for the fictitious Tailspin company.
2. Register the Surveys application and the backend web API with Azure AD.
3. Create an Azure Redis Cache instance.
4. Configure application settings and create a local database.
5. Run the application and sign up a new tenant.
6. Add application roles to users.

## Prerequisites

- Visual Studio 2022 installed
- [Microsoft Azure](https://azure.microsoft.com/en) account
- [Entity Framework Core tools](https://docs.microsoft.com/en-us/ef/core/cli/dotnet#installing-the-tools)

## Create the Tailspin tenant

Tailspin is the fictitious company that hosts the Surveys application. Tailspin uses Azure AD to enable other tenants to register with the app. Those customers can then use their Azure AD credentials to sign into the app.

In this step, you'll create an Azure AD directory for Tailspin.

1. Sign in to the [Azure portal][portal].

2. In the left pane, select [Create a resource][createhub].

3. Select the **Identity** category, and then select the **Azure Active Directory** resource type.

   ![Screenshot of create resource](./images/running-the-app/create-aad-resource.png)

4. Enter `Tailspin` for the organization name, and a domain name. The domain name will have the form `xxxx.onmicrosoft.com` and must be globally unique.

   ![Screenshot of create tenant](./images/running-the-app/create-tenant.png)

5. Select the **Create** button to create the new Tenant.

To complete the end-to-end scenario, you'll need a second Azure AD directory to represent a customer that signs up for the application. You can use your default Azure AD directory (not Tailspin), or create a new directory for this purpose. In the examples, we use Contoso as the fictitious customer.

## Register the Surveys web API

1. In the [Azure portal][portal], switch to the newly created Tailspin directory by selecting your account in the top right corner of the portal and choosing **Switch Directory**, then select **Tailspin** from the directory list.

2. In the left pane, choose [**Azure Active Directory**][activedirectorypane], then select [**App Registrations**][aad-appregistrations]

3. Select **New registration**.

4. Enter the following information:

   - **Name**: `Surveys.WebAPI`
   - **Supported Account Types**: `Accounts in any organizational directory (Any Azure AD directory - Multitenant)`
   - **Application type**: `Web`
   - **Redirect URI**: `https://localhost:44301/`

   ![Screenshot for registering the Web API](./images/running-the-app/register-web-api.png)

5. Select **Register**.

### Expose API settings

1. In [**App Registrations**][aad-appregistrations] , select the **Surveys.WebAPI** application.

2. Select **Expose an API** > Application ID URI **Set**.

3. In the **Set the App ID URI** edit box, enter `https://<domain>/surveys.webapi`, where `<domain>` is the domain name of the directory. For example: `https://tailspin.onmicrosoft.com/surveys.webapi`

   ![Screenshot for registering the Web API](./images/running-the-app/register-webapi-appuri.png)

4. Select **Save**.

5. Select **Add a scope** to define an API access scope that will later be used to authorize clients. Enter the following information:

   - **Scope name**: `surveys.access`
   - **who can consent**: `admins and users`
   - **Admin consent display name**: `Web API access`
   - **Admin consent description**: `Allows access to the Tailspin Web API`
   - **User consent display name**: `Web API access`
   - **User consent description**: `Allows access to the Tailspin Web API`
   - **State**: `enabled`

     ![Screenshot adding a Web API scope](./images/running-the-app/webapi-add-scope.png)

6. Click **Add scope**

## Register the Surveys web app

1. In [**App Registrations**][aad-appregistrations], select **New Registration**.

2. Enter the following information:

   - **Name**: `Surveys`
   - **Supported Account Types**: `Accounts in any organizational directory (Any Azure AD directory - Multitenant)`
   - **Application type**: `Web`
   - **Redirect URI**: `https://localhost:44300/`

   Notice that the sign-on URL has a different port number from the `Surveys.WebAPI` app in the previous step.

3. Select **Register**.

### Set the Application ID URI

1. In [**App Registrations**][aad-appregistrations], select the new **Surveys** application. Copy the application ID. You will need this later.

2. Select **Expose an API** > Application ID URI **Set**

3. In **Set the App ID URI**, enter `https://<domain>/surveys`, where `<domain>` is the domain name of the directory.

4. Select **Save**.

### Set Authentication options

1. Select **Authentication**

2. Under **Redirect URIs**, select **Add URI** and enter: `https://localhost:44300/signin-oidc`.

3. Under **Implicit grant**, select **ID Tokens**

4. Select **Save**.

### Configure client secrets

1. Select **Certificates and Secrets**

2. Under **Client secrets**, select **New client secret**.

3. Enter a description, such as `client secret`.

4. Under **Expires**, select **In 1 year**.

5. Select **Add**. The client secret will be generated at this point.

6. Before you navigate away from this blade, copy the value of the secret.

   > The value of the secret will not be visible again after you navigate away from the blade.

### Set API permissions

1. Select **API Permissions**, then select **Add a permission**.

2. In **Request API Permissions** select **My APIs**.

3. Select `Surveys.WebAPI`.

4. Under **What type of permissions does your application require?**, select **Delegated Permissions**.

5. Under **Select Permissions**, select the **survey.access** scope.

   ![Screenshot for adding scope permissions](./images/running-the-app/request-api-permission.png)

6. Select **Add permissions**

## Update the application manifests

1. In [**App Registrations**][aad-appregistrations] , select the **Surveys.WebAPI** application.

2. Select **Manifest**.

3. Add the following JSON to the `appRoles` element. Generate new GUIDs for the `id` properties.

   ```json
   {
     "allowedMemberTypes": ["User"],
     "description": "Creators can create surveys",
     "displayName": "SurveyCreator",
     "id": "<Generate a new GUID. Example: 1b4f816e-5eaf-48b9-8613-7923830595ad>",
     "isEnabled": true,
     "value": "SurveyCreator"
   },
   {
     "allowedMemberTypes": ["User"],
     "description": "Administrators can manage the surveys in their tenant",
     "displayName": "SurveyAdmin",
     "id": "<Generate a new GUID>",
     "isEnabled": true,
     "value": "SurveyAdmin"
   }
   ```

4. In the `knownClientApplications` property, add the application ID for the Surveys web application, which you got when you registered the Surveys application earlier. For example:

   ```json
   "knownClientApplications": ["be2cea23-aa0e-4e98-8b21-2963d494912e"],
   ```

   This setting adds the Surveys app to the list of clients authorized to call the web API.

5. Set `accessTokenAcceptedVersion` to 2, we are going to accept only access token v2.0

   ```json
   "accessTokenAcceptedVersion": 2,
   ```

6. Select **Save**.

Now repeat the same steps for the Surveys app, except do not add an entry for `knownClientApplications`. Use the same role definitions, but generate new GUIDs for the IDs.

## Create a new Redis Cache instance

The Surveys application uses Redis to cache OAuth 2 access tokens. To create the cache:

1. In the [Azure portal][portal]

2. In the left pane, select [Create a resource][createhub].

3. Select **Databases** > **Azure Cache for Redis**.

4. Enter the required information, including DNS name, resource group, location, and pricing tier. You can create a new resource group or use an existing resource group.

5. Select **Create**.

6. After the Redis cache is created, navigate to the resource in the portal.

7. Click **Access keys** and copy the primary key.

For more information about creating a Redis cache, see [Quickstart: Use Azure Cache for Redis with a .NET Framework application](https://docs.microsoft.com/azure/azure-cache-for-redis/cache-dotnet-how-to-use-azure-redis-cache).

## Set application secrets

1. Open the Tailspin.Surveys solution in Visual Studio.

2. In Solution Explorer, right-click the Tailspin.Surveys.Web project and select **Manage User Secrets**.

3. In the secrets.json file, paste in the following:

   ```json
   {
     "AzureAd": {
       "Instance": "https://login.microsoftonline.com/",
       "Domain": "<domain>.onmicrosoft.com",
       "TenantId": "common",
       "ClientId": "<Surveys application ID>",
       "ClientSecret": "<Surveys app client secret>",
       "CallbackPath": "/signin-oidc",
       "SignedOutCallbackPath ": "/signout-callback-oidc",
       "PostLogoutRedirectUri": "https://localhost:44300/",
       "WebApiResourceId": "<Surveys.WebAPI app ID URI>"
     },
     "Redis": {
       "Configuration": "<Redis DNS name>.redis.cache.windows.net,password=<Redis primary key>,ssl=true"
     }
   }
   ```

   Replace the items shown in angle brackets, as follows:

   - `AzureAd:Domain`: The AD primary domain.
   - `AzureAd:ClientId`: The application ID of the Surveys app.
   - `AzureAd:ClientSecret`: The key that you generated when you registered the Surveys application in Azure AD.
   - `AzureAd:WebApiResourceId`: The App ID URI that you specified when you created the Surveys.WebAPI application in Azure AD. It should have the form `https://<directory>.onmicrosoft.com/surveys.webapi`
   - `Redis:Configuration`: Build this string from the DNS name of the Redis cache and the primary access key. For example, "tailspin.redis.cache.windows.net,password=2h5tBxxx,ssl=true".

4. Save the updated secrets.json file.

5. Repeat these steps for the Tailspin.Surveys.WebAPI project, but paste the following into secrets.json. Replace the items in angle brackets, as before.

   ```json
   {
     "AzureAd": {
       "WebApiResourceId": "<Surveys.WebAPI app ID URI>",
       "Instance": "https://login.microsoftonline.com/",
       "Domain": "<domain>.onmicrosoft.com",
       "TenantId": "common",
       "ClientId": "<Surveys application ID>",
     },
     "Redis": {
       "Configuration": "<Redis DNS name>.redis.cache.windows.net,password=<Redis primary key>,ssl=true"
     }
   }
   ```

## Set application settings 

1. Open the appsettings.json file in Tailspin.Surveys.Web

2. Paste the following:


   ```json
   {
    "SurveyApi": {
    "BaseUrl": "https://localhost:44301",
    "Scopes": "https://<domain>.onmicrosoft.com/surveys.webapi/surveys.access",
    "Name": "SurveyApi"
    },
    "Data": {
    "SurveysConnectionString": "Server(localdb)\\MSSQLLocalDB;Database=Tailspin.SurveysDB;Trusted_Connection=True;MultipleActiveResultSets=true"
    },
    "KeyVaultName": "See Optional step in later"
   }
   ```

Replace the items shown in angle brackets, as follows:

  - `Scopes`:  Add the domain name. This is the scope created earlier.
  - `KeyVaultName`:  Add the domain name. This is the scope created earlier.

## Initialize the database

In this step, you will use Entity Framework 7 to create a local SQL database, using LocalDB.

1. Open a command window

2. Navigate to the Tailspin.Surveys.Data project.

3. Run the following command:

   ```bat
   dotnet ef database update --startup-project ..\Tailspin.Surveys.Web
   ```
   


## Run the application

To run the application, start both the Tailspin.Surveys.Web and Tailspin.Surveys.WebAPI projects.

You can set Visual Studio to run both projects automatically on F5, as follows:

1. In Solution Explorer, right-click the solution and click **Set Startup Projects**.
2. Select **Multiple startup projects**.
3. Set **Action** = **Start** for the Tailspin.Surveys.Web and Tailspin.Surveys.WebAPI projects.

## Sign up a new tenant

When the application starts, you are not signed in, so you see the welcome page:

![Welcome page](./images/running-the-app/screenshot1.png)

To sign up an organization:

1. Click **Enroll your company in Tailspin**.
2. Sign in to the Azure AD directory that represents the organization using the Surveys app. You must sign in as an admin user.
3. Accept the consent prompt.

The application registers the tenant, and then signs you out. The app signs you out because you need to set up the application roles in Azure AD, before using the application.

![After sign up](./images/running-the-app/screenshot2.png)

## Assign application roles

When a tenant signs up, an AD admin for the tenant must assign application roles to users.

1. In the [Azure portal][portal], switch to the Azure AD directory that you used to sign up for the Surveys app.

2. In the left pane, choose [**Azure Active Directory**][activedirectorypane], then select [**Enterprise Applications**][aad-enterprise-applications]. You should see the applications `Survey` and `Survey.WebAPI` listed. If not, make sure that you completed the sign up process.

3. Select the **Surveys** application.

4. Select **Users and Groups** > **Add user**.

5. If you have Azure AD Premium, click **Users and groups**. Otherwise, click **Users**. (Assigning a role to a group requires Azure AD Premium.)

6. Select one or more users and click **Select**.

   ![Select user or group](./images/running-the-app/select-user-or-group.png)

7. Select the role and click **Select**.

   ![Select user or group](./images/running-the-app/select-role.png)

8. Click **Assign**.

Repeat the same steps to assign roles for the Survey.WebAPI application.

> A user should always have the same roles in both Survey and Survey.WebAPI. Otherwise, the user will have inconsistent permissions, which may lead to 403 (Forbidden) errors from the Web API.

Now go back to the app and sign in again. Click **My Surveys**. If the user is assigned to the SurveyAdmin or SurveyCreator role, you will see a **Create Survey** button, indicating that the user has permissions to create a new survey.

![My surveys](./images/running-the-app/screenshot3.png)

## Optional: Enable Key Vault

As a security best practice, you should never store application secrets such as connection strings in source control. To enable storing secrets in Key Vault, follow the steps [here](./key-vault.md).

<!-- links -->

[portal]: https://portal.azure.com
[activedirectorypane]: https://ms.portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/Overview
[aad-appregistrations]: https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps
[aad-enterprise-applications]: https://portal.azure.com/#blade/Microsoft_AAD_IAM/StartboardApplicationsMenuBlade/AllApps
[createhub]: https://ms.portal.azure.com/#create/hub
[vs2017]: https://www.visualstudio.com/vs/
[appregistrations]: https://ms.portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade
