# Deploy your SAP BTP Application

## Prepare Your Project Configuration for Cloud Foundry Deployments

Make some adjustments to ensure that the application can be deployed to SAP BTP Cloud Foundry runtime as a central launchpad component.

1. Configure the web app to connect to the app services: To define a route in the web app, open the file [*./app/poetryslammanager/xs-app.json*](../../../tree/main-single-tenant/app/poetryslammanager/xs-app.json) and add the following route at position 1 (Note that the order matters: The most specific one must come first.)

     >  Note: You'll define the destination *launchpad* later as part of project configuration *mta.yml* file. 

  ```json
    {
      "authenticationType": "none",
      "csrfProtection": false,
      "source": "^/odata/v4/poetryslammanager/",
      "destination": "launchpad"
    }
  ```

2. In the file [*./app/poetryslammanager/xs-app.json*](../../../tree/main-single-tenant/app/poetryslammanager/xs-app.json), you can also define a Content Security Policy (CSP). The CSP allows you to restrict which resources can be loaded.
   > Note: This is just an example of a CSP, we recommend that you check your use case and your security settings carefully.

  ```json  
    "responseHeaders": [
      {
        "name": "Content-Security-Policy",
        "value": "default-src 'self' https://sapui5.hana.ondemand.com; frame-ancestors 'self' https://*.hana.ondemand.com;  object-src 'none';"
      }
    ]
  ```

> Note: If you copied the *manifest.json* during the development of the core application, you can skip the next two steps.

3. Add the service name to the web app configuration file [*./app/poetryslammanager/webapp/manifest.json*](../../../tree/main-single-tenant/app/poetryslammanager/webapp/manifest.json). 

    > Note: This service name must be unique within your account and will appear in the runtime URL of the application.

  ```json
  "sap.cloud": {
    "service": "poetryslammanager",
    "public": true
  }
  ```

4. In the *dataSources* section, adopt the `uri` by removing the `/` before `odata`. The URI of the data sources is changed now to a relative path.

  ```json
  "dataSources": {
    "mainService": {
      "uri": "odata/v4/poetryslammanager/",
        "type": "OData",
        "settings": {
        "annotations": [],
        "odataVersion": "4.0"
      }
    }
  }
  ```

5. In the file [*./app/poetryslammanager/ui5-deploy.yaml*](../../../tree/main-single-tenant/app/poetryslammanager/ui5-deploy.yaml), the packaging of the web UI application is defined. Replace the line `afterTask: replaceVersion` with `beforeTask: generateCachebusterInfo` to ensure that the package is created correctly for the deployment.

## Rename the Application in the Config Files

The wizard for creating a new SAP Cloud Application Programming Model project generated the *mta.yaml* file, which contains all deployment-relevant information. 

The application had been created with the name *partner-reference-application*, but the module and service names in the [*./mta.yaml*](../../../tree/main-single-tenant/mta.yaml) file were renamed from `partner-reference-application-*` to `poetry-slams-*`.

Additionally, rename `partner-reference-application` in the *undeploy* step of the [*./package.json*](../../../tree/main-single-tenant/package.json) to `poetry-slams`.

## Configure the Project Deployment Config Files

Adjust the [*./mta.yaml*](../../../tree/main-single-tenant/mta.yaml) file. 

1. Enhance the build parameters to install the node modules before the build with the CDS development kit.
```yml
  build-parameters:
    before-all:
    - builder: custom
      commands:
      - npm ci --production
      - npx -p @sap/cds-dk cds build --production
  ```

2. Add and configure the destination content module. This is where you define destinations and service keys for the destinations. After the MTA app has been deployed, you'll see two destinations *html_repo_host* and *…_uaa_fiori* in your subaccount.

    > Note that the service name *poetryslammanager* of the web app must match the service name used in the file *manifest.json* of the web app.

  ```yml
  modules:
  - name: poetry-slams-destination-content
    type: com.sap.application.content
    requires:
    - name: poetry-slams-repo-host
      parameters:
        service-key:
          name: poetry-slams-repo-host-key
    - name: poetry-slams-auth
      parameters:
        service-key:
          name: poetry-slams-auth-key
    - name: poetry-slams-destination-service
      parameters:
        content:
          instance:
            destinations:
            - Name: poetryslammanager-repo-host-dest
              ServiceInstanceName: poetry-slams-html5-srv
              ServiceKeyName: poetry-slams-repo-host-key
              sap.cloud.service: poetryslammanager
            - Authentication: OAuth2UserTokenExchange
              Name: poetryslammanager-uaa-fiori-dest
              ServiceInstanceName: poetry-slams-auth
              ServiceKeyName: poetry-slams-auth-key
              sap.cloud.service: poetryslammanager
            existing_destinations_policy: update
        content-target: true
    build-parameters:
      no-source: true
  ```

3. Adjust the destination service resource in the file *mta.yaml*. Define the destination resource for the route as defined in the web application configuration file *./app/poetryslammanager/xs-app.json*.

    > Note that the name *launchpad* of the route in *xs-app.json* must match the destination service resource name in the file *mta.yaml*. Enable the HTML5 runtime and add the service API as a required dependency of the destination:

  ```yml
  resources:
  - name: poetry-slams-destination-service
    type: org.cloudfoundry.managed-service
    parameters:
      config:
        HTML5Runtime_enabled: true
        init_data:
          instance:
            destinations:
            - Authentication: NoAuthentication
              Name: ui5
              ProxyType: Internet
              Type: HTTP
              URL: https://ui5.sap.com
            - Authentication: NoAuthentication
              HTML5.DynamicDestination: true
              HTML5.ForwardAuthToken: true
              Name: launchpad
              ProxyType: Internet
              Type: HTTP
              URL: ~{srv-api/srv-url}
            existing_destinations_policy: update
        version: 1.0.0
      service: destination
      service-name: poetry-slams-destination-service
      service-plan: lite
    requires:
    - name: srv-api
  ```
  
4. After you've applied the changes described above in the file *mta.yml*, this is what the file looks like: [the MTA file of the sample application](../../../tree/main-single-tenant/mta.yaml).

    > Note that a correct indentation is required.

## Adopt NPM Modules
The CDS libraries are offered as npm modules in the *package.json*. After the creation of the SAP Cloud Application Programming Model (CAP) project via the wizard, the npm @sap/cds-dk module is added to the [*./package.json*](../../../tree/main-single-tenant/package.json) as a development dependency. This module needs to be moved to the *dependencies* section. You can check the [*package.json*](../../../tree/main-single-tenant/package.json) of the sample implementation to view the result.

## Deploy to Cloud Foundry

1. Open a new terminal and log on to SAP BTP Cloud Foundry runtime: 
	1. Run the command `cf login`. 
	2. Enter the SAP BTP Cloud Foundry runtime API of your environment (for example, `https://api.cf.eu10.hana.ondemand.com`).
	3. Enter your development user and password.
	4. Select org of the SAP BTP provider subaccount for the application (*poetryslams*). 
	5. Select the SAP BTP Cloud Foundry runtime space (*runtime*).

2. Run the command `npm install` to install the messaging npm packages.

3. Run the command `npm run build` to build the project. The *archive.mtar* is added to the folder *mta_archives*. 

4. To deploy the application, run the command `npm run deploy`.

Looking for more details? Go to the [SAP Cloud Application Programming Model documentation on how to deploy to SAP BTP Cloud Foundry runtime](https://cap.cloud.sap/docs/guides/deployment/to-cf).

## Test the HTML5 Application

1. To test your application, navigate to *HTML5 Applications* in the SAP BTP cockpit and choose *poetryslammanager*. 
2. The application opens and the launchpad is displayed with one tile. 
3. As you have not yet set up any authorizations, the application will tell you that you're not authorized to use it when you click on the tile.

## Configure SAP Build Work Zone

Since the web application is now available as an HTML5 application, it's ready to be added to SAP Build Work Zone.

To open the *Site Manager*, launch the application *SAP Build Work Zone, standard edition* *Instance and Subscriptions* in your SAP BTP subaccount.

### Fetch the Latest Version of Your Web Application

1. Open the *Channel Manager*. The *HTML5 Apps* content channel is created automatically and all web applications that you deployed to the SAP BTP subaccount are automatically added as content to this content provider.

2. In the *HTML5 Apps* content channel, choose *Update content* to fetch any updates of your web application. The *HTML5 Apps* content channel now exposes the latest version of the web application. 

> Note: You must update the content channel every time you made changes to the web application.

### Add the Web Application to Your Content

1. Open the *Content Manager*.
2. Go to the *Content Explorer* sheet.
3. Select the content provider *HTML5 Apps*.
4. To add your web application to your content, choose *Add*. 

### Create a Group and Add Your App 

1. Open the *Content Manager*.
2. Create a new group and enter a title and description.
3. On the *Apps* tab, you see a list of available apps. Move the red slider in the *Assignment Status* column of your app to assign your app to the group. The color of the slider changes to green.
4. Save your changes.

### Assign the Web Application to the Default Role

In this step, you assign your app to the *Everyone* role, which is a default role. The content assigned to this role is visible to all users.

1. Open the *Content Manager*.
2. To open the *Role Editor*, choose *Everyone*.
3. Choose *Edit*.  
4. On the *Apps* tab, you can see a list of available apps. Move the red slider in the *Assignment Status* column of your app to assign your app to the group. The color of the slider changes to green.
5. Save your changes.

### Create and/or Update a Site

In this step, you create and review a launchpad site. If you already have a site, just add your web application.

1. Open the *Site Directory*. 
2. Create a site and enter a site name.
3. To launch the site, open the *URL* provided in the *Properties* of the *Site Settings*.
4. Test your web application. 

> Note: Note down the site URL as **SAP BTP Application Launchpad URL**. On the launchpad, open the context menu of the *Poetry Slam Manager* tile and note down the URL as **SAP BTP Application URL**.

## Configure Authentication and Authorization 

You use the Identity Authentication service as a corporate identity provider (IdP) and establish a trust relationship between the service provider (the SAP BTP subaccount to which you deployed the application) and the Identity Authentication service tenant. As a result, the SAP BTP subaccount and the application delegate user authentications to the Identity Authentication service tenant including single sign-on. Furthermore, you use the Identity Authentication service tenant to assign authorization roles to users via user groups.

However, as a prerequisite, you must have admin access to an Identity Authentication service tenant. 

### Configure Single Sign-On Using the Identity Authentication Service

As a preferred approach, you configure trust between the SAP BTP subaccount and the Identity Authentication service using OpenID Connect (OIDC). As a fallback option, a SAML 2.0 trust configuration is described as well.

#### OpenID Connect Configuration

Set up the trust relationship between the SAP BTP subaccount to the Identity Authentication service using OpenID Connect (OIDC).

> Note: As a prerequisite for this setup, the SAP BTP subaccount and the Identity Authentication service tenant must be assigned to the same customer ID.

1. Within your SAP BTP subaccount, open the menu item *Security* and go to *Trust Configuration*. 
2. Choose *Establish Trust* and select the Identity Authentication service tenant to set up the OIDC trust configuration.
3. On the Identity Authentication service admin UI, log on to the Identity Authentication service admin UI (URL: [IAS]/admin/). 
4. Open the menu item *Applications* and search for the application that refers to your SAP BTP subaccount
  > Note that the name typically follows the pattern: *XSUAA_[subaccount-name]*.
5. Edit the application and change the following fields:
    - The display name appears on the user log-on screen and the login applies to all applications linked to the Identity Authentication service tenant (following the single-sign on principle). Change the *Display Name* to something meaningful from an end-user perspective representing the scope of the Identity Authentication service.
    - Enter the *Home URL*, for example, the link to the SAP Build Work Zone launchpad or the application.
	
#### SAML 2.0 Configuration (Fallback)

Set up the trust relationship between the SAP BTP subaccount to the Identity Authentication service using SAML 2.0. This approach is the fallback trust configuration if the OIDC configuration is not possible. 
	
> Note: This fallback applies only if the SAP BTP subscriber subaccount and the Identity Authentication service tenant are not assigned to the same customer ID. This setup comes with limitations regarding remote access to the OData services of the SAP BTP app with principal propagation.

1. Within your SAP BTP subaccount, to download the *Service provider SAML metadata* file, open the menu item *Security* and go to *Trust Configuration*. 
2. Choose *Download SAML Metadata*.
3. On the Identity Authentication service Admin UI, open the menu item *Applications* and create a new application of the type *SAP BTP solution*:
	1. Enter the required information such as application display name, application URL, and so on. The display name appears on the user log-on screen and the login applies to all applications linked to the Identity Authentication service tenant (following the single-sign on principle). Choose something meaningful from an end-user perspective representing the scope of the Identity Authentication service.
	2. Open the *SAML 2.0 Configuration* section and upload the *Service provider SAML metadata* file from the SAP BTP subaccount.
	3. Open the *Subject Name identifier* section and select *E-Mail* as basic attribute.
	4. Open the *Default Name ID Format* section and select *E-Mail*.
4. To download the *IDP SAML metadata file*: 
	1. Open the menu item *Tenant Settings* and go to *SAML 2.0 Configuration*.
	2. Choose *Download Metadata File*.
5. Within your SAP BTP subaccount, open the menu item *Security* and go to *Trust Configuration*.
6. Choose *New SAML Trust Configuration*. 
7. Upload the *IDP SAML metadata* file and enter a meaningful name and description for the Identity Authentication service (for example, `Corporate IDP` or `Custom IAS (SAML2.0)`).
	
### Set Up Users and User Groups

In this example, you use Identity Authentication service user groups to assign authorizaton roles to users. The user groups will be passed as *assertion attribute* to the SAP BTP subaccount and will be mapped to the respective role collections in the SAP BTP subaccount. 

1. On the Identity Authentication service Admin UI, open the menu item *User Management* and add the users that should have access to the SAP BTP application. Enter user details such as name and e-mail. But take into account that the e-mail is used as the identifying attribute. As a recommendation, use the e-mail address that is used in the ERP system that you'll integrate later.
2. Open the menu item *Groups* and add user groups that represent typical user roles. Enter a unique (technical) *Name* and a meaningful *Display Name*, for example:

    | Name                      | Display Name              |
    | :------------------------ | :------------------------ |
    | `Poetry_Slam_Manager`     | `Poetry Slam Manager`     |
    | `Poetry_Slam_Visitor`     | `Poetry Slam Visitor`     |

3. Open the menu item *Applications*, open the application referring to the SAP BTP subaccount with your application, and navigate to *Attributes*.
4. Check if there is an attribute with the name *Groups* and value *Groups*. If not, add the attribute mapping accordingly.
	> Note: Capital letters are required to ensure a correct mapping.
5. Within your SAP BTP subaccount, open the menu item *Role Collections* and add the user groups (using the unique technical name of the user group) to the role collections that you want to assign to the respective users with the user group:

    | Role Collection                    | User Groups            |
	  | :--------------------------------- | :--------------------- |
	  | `PoetrySlamManagerRoleCollection`  | `Poetry_Slam_Manager`  |
	  | `PoetrySlamVisitorRoleCollection`  | `Poetry_Slam_Visitor`  |

### Log On to the SAP BTP Application and Test Single Sign-On

Launch your SAP BTP application and select the Identity Authentication service tenant as IdP. 

> Note: If the user has not yet been replicated from the Identity Authentication service tenant to the SAP BTP subaccount, the first attempt to open the app may fail with an authorization error message (at the very latest, the replication is triggered and executed automatically at this point). The second login attempt to open the app will be successful.

You may deactivate the *Default Identity Provider* (which refers to the SAP ID Service) in the trust center of your SAP BTP subaccount.

Looking for more information on the functionality of Poetry Slam Manager, the sample application? Go to the [guided tour](17-Guided-Tour.md).