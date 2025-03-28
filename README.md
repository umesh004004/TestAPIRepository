# TestAPIRepository
TestAPIRepository Umesh Laxkar

Kyma ❌
Cloud Foundry ✅

Important - This part of the tutorial is required for Cloud Foundry deployments only!

As your SaaS application contains an API that allows the SaaS consumers to interact programmatically with their tenant database containers, you need to ensure that your API endpoints are properly managed and monitored. For this purpose, you should implement features like rate limiting to prevent e.g., DoS attacks. Furthermore, you can ensure fair usage of the resources among your consumers by e.g., setting up a quota depending on the chosen plan. A premium consumer might be eligible to send more requests per second than a standard consumer. Proper monitoring of your API will help you to analyze performance issues and to identify problems of your consumers.

In this part of the mission, you will learn how to ensure that each and every request to your SaaS API is first going through SAP API Management, which will be the dedicated standard solution provided by SAP for all API-related requirements.

Integrate SAP API Management
1. Architecture
2. Prerequisites
3. APIM as route service
4. Bind the route service
5. API Policy Deep Dive
5.1. Decode the JWT token
5.2. Spike Arrest Policy
5.3. API Quota Policies
5.5. Update API Proxy and deploy changes
6. Test the setup
7. Deployment Descriptor
8. Further Information
1. Architecture
SAP API Management is a new component in the central part of the Advanced Version architecture. While in the Basic Version, the API calls were directly accessing the CAP-based API service, now all requests will be passing through this additional component, giving you great flexibility in how to handle your in and outbound API traffic.

See the relevant part of the solution architecture below (click to enlarge):



2. Prerequisites
For this setup, please make sure you have an SAP API Management instance up and running. As SAP API Management is a capability of SAP Integration Suite, please subscribe to SAP Integration Suite and activate the respective API Management feature.

Check the following SAP Help documentations to find detailed step-by-step guide.

SAP Help - Setting Up API Management Capability from Integration Suite
Tutorial Navigator - Set Up API Management from Integration Suite


Please ensure the API Management capability of your SAP Integration Suite instance is successfully enabled before continuing with any of the next steps!

3. APIM as route service
To connect your SaaS API with SAP API Management, you can use an SAP BTP service called "APIM-as-route-service", which is also explained in greater detail by the following blog posts click here and here. Further information about route services can be found in the official Cloud Foundry documentation click here.

Combining this service instance with your API route allows you to enforce API policies like Spike Arrest or Quotas no matter whether the accessing client is calling the API Proxy URL or the standard route of your API. So please set up an instance of the apim-as-a-route-service offering as you can see in the following screenshot.

Hint - If you cannot find the respective service plan, make sure you assigned it in your subaccount entitlements! The service plan is part of the API Management, API portal service!



As SAP Integration Suite is one of the most powerful but also quite expensive SaaS products, you might consider the usage for your productive SaaS environment only.

4. Bind the route service
Now you need to bind the route service to your standard SaaS API route. This can be done using the cf CLI command bind-route-service (or brs).

Windows (Command Line)

cf brs `<API service domain>` `<route service>` --hostname `<API service hostname>` -c '{\"api_name\":\"<API-Proxy name>\"}'

# Example #
cf brs cfapps.eu10.hana.ondemand.com susaas-apim-route-service --hostname dev-susaas-api-srv -c '{\"api_name\":\"SusaaS-API-Proxy\"}'
Mac & Windows (Power Shell)

cf brs `<API service domain>` `<route service>` --hostname `<API service hostname>` -c "{\"api_name\":\"<API-Proxy name>\"}"

# Example #
cf brs cfapps.eu10.hana.ondemand.com susaas-apim-route-service --hostname dev-susaas-api-srv -c "{\"api_name\":\"SusaaS-API-Proxy\"}"
route service - The name of your route-service instance created in the last step.
API-Proxy name - You're free to choose a name your choice for the API-Proxy which will be created in API Management.
API service domain - The domain of your API service like cfapps.eu10.hana.ondemand.com or your custom domain.
API service hostname - The hostname of your API service returned by the cf apps CLI command.

Important - The command might differ depending on your cf CLI version. You can use cf brs --help to find the correct command related to your cf CLI version. Also make sure to change the change the API service domain in case of Custom Domain usage.

After successfully running this command in your cf CLI, you will see a new API Proxy called SusaaS-API-Proxy in your SAP API Management Develop menu.



Checking the API Proxy details you will see in the description that this API Proxy is bound to a Cloud Foundry application which is your SaaS API. From now on, all requests reaching your SaaS API route will be running through SAP API Management, and respective API policies are applied.



5. API Policy Deep Dive
You can now apply relevant API Policies to your API Proxy using preconfigured templates and features. In this sample, you will learn how to set up a Spike Arrest component for rate limiting and different quotas based on the plan (standard/premium) subscribed by the consumer.

To set up the respective policies, click on Policies in the top right of your API Proxy.



Switch to edit mode by clicking on Edit in the Policy Editor.



5.1. Decode the JWT token
Let us start with the PreFlow of our API Proxy, in which we place the Rate Limiter of our SaaS API. Check the following SAP Help documentation to learn more about the Flow types in SAP API Management.

5.1.1. In this use-case, we will distinguish our SaaS API clients by their unique Client ID, which can be found in the JWT token of each request. Therefore, please first add a feature called DecodeJWT which will allow you to make use of the JWT token content in subsequent steps.



5.1.2. Rename the flow element decodeJwt. If you choose a different name, please keep track of it, as you will need it in one of the next steps.



5.1.3. Please remove the <Source>var.jwt</Source> line from the configuration.



5.2. Spike Arrest Policy
The Spike Arrest Policy allows you to throttle the number of requests processed by your API Proxy. It protects you against performance lags as well as downtimes and is an essential component of each enterprise-ready API Proxy.

5.2.1. After decoding the JWT token, we can make use of the Spike Arrest feature in our policies toolbox. Please add a new instance and rename the flow element spikeArrest.



5.2.2. The Spike Arrest instance requires a Client identifier which is the Client Id in our scenario. The Client Id is origination from the previous flow element (used to decode the JWT token). You might need to change the decodeJwt value in case you named your initial flow element differently.

<Identifier ref="jwt.decodeJwt.claim.client_id"/>
5.2.3. Feel free to adapt the number of requests which you want to allow per minute (pm) or second (ps). Check the documentation click here of Spike Arrest as the configuration options are extremely comprehensive.

<Rate>1ps</Rate> instead of <Rate>30pm</Rate>


5.3. API Quota Policies
Besides an API Rate Limit (protecting our SaaS API from DoS attacks), we enhance our API Proxy by introducing a plan-based Quota. This allows us to differentiate standard from premium customers, offering different service levels (like number of requests per day) for the SaaS API.

5.3.1. To differentiate between the different API service plans, we are using Subflows. Those Subflows will be executed based on a certain condition, after the PreFlow was executed.

Just create a new Subflow for the Proxy Endpoint, by clicking on the + icon.



5.3.2. We use three Subflows called standardPlanFlow, premiumPlanFlow and trialPlanFlow. These Subflows will be executed right after the generic PreFlow, depending on conditions you define. The PreFlow is executed for all requests.



5.3.3. Once again, we differentiate our requests by using the decoded JWT token data. Our SaaS API Service Broker ensures, that the selected service plan is injected as a scope to all issued JWT tokens. This allows you to read the service plan details and to include it in the Flow Condition as follows:

Hint - Please make sure to select the standardPlanFlow before adding the condition.

jwt.decodeJwt.claim.scope ~ "*plan_standard"



5.3.4. For the premiumPlanFlow the condition looks as follows.

jwt.decodeJwt.claim.scope ~ "*plan_premium"

5.3.5. For the trialPlanFlow the condition looks as follows.

jwt.decodeJwt.claim.scope ~ "*plan_trial"

Hint - Reading the service plan from the JWT token scopes might be improved by a different approach in the future.

5.3.6. Now requests will be handled by the different Subflows depending on the consumer's service plan selection. In those Subflows, we can define different Quota allowances. To add a very simple Quota limit to the API, we use the Quota feature from the policies toolbox. Please name the new flow elements quotaStandard in the standard, quotaPremium in the premium and quotaTrial in the trial flow.



5.3.7. For our sample application, the standard and trial quota is configure as below. This configuration allows API customers exactly 1200 daily requests to your API. The comprehensive configuration options of the Quota policy can be found in the respective documentation click here.

Hint - For the premium plan, you we double the number of daily requests to 2400, but feel free to update it to a configuration of your choice.

Important - Please ensure, the Quota policy configuration once again contains the Client Id identifier as in the following sample.

Important - Please make sure not to provide any value into the Condition String field of the Quota policy (See screenshots below).

<Quota async="false" continueOnError="false" enabled="true" type="calendar" xmlns="http://www.sap.com/apimgmt">
 	<Identifier ref="jwt.decodeJwt.claim.client_id"/>
 	<Allow count="1200"/>
 	<Interval>1</Interval>
	<Distributed>true</Distributed>
 	<StartTime>2015-2-11 12:00:00</StartTime>
	<Synchronous>true</Synchronous>
 	<TimeUnit>day</TimeUnit>
</Quota>


5.3.8. Double check the Condition Strings of your Subflows and the respective quota configurations, which should resemble the following screenshots.

 

 

 

5.5. Update API Proxy and deploy changes
5.5.1. Please click on Update in the upper right of your Policy Editor.



5.5.2. Save your API Proxy changes by clicking on Save.



5.5.3. Make sure to Deploy the latest version of your API Proxy by selecting the respective option.



6. Test the setup
That's it, you've successfully integrated your SaaS API with SAP API Management and already configured some API Policies. To test the setup, feel free to create a new service key in a consumer subaccount (or use an existing one) and try calling your API endpoint e.g., using the sample HTTP files.

You will notice that calling the API more than once per second (e.g. using Postman or the HTTP files), will result in an error message sent by SAP API Management as the Spike Arrest policy will jump in.



7. Deployment Descriptor
The API Management as route service (sap-apim-route-service) instance can also be defined in your mta.yaml deployment descriptor. Just add the following code snippet to your resources. Still, you will need to execute the cf bind-route-service command as described in this tutorial.

resources:
  - name: susaas-api-route-service
    type: org.cloudfoundry.managed-service
    parameters:
      service: apimanagement-apiportal
      service-plan: apim-as-route-service
8. Further Information
Please use the following links to find further information on the topics above:

SAP Help - SAP Integration Suite
SAP Help - SAP API Management
SAP Help - SAP API Management in the Cloud Foundry Environment
Cloud Foundry CLI Reference Guide - v8
Cloud Foundry CLI Reference Guide - v7
SAP Blog - SAP APIM - Route Service to Manage Cloud Foundry Apps
SAP Blog - Service Plan of SAP API Management in Cloud Foundry Environment
Cloud Foundry Documentation - Route Services
apigee Documentation - Policy reference overview
apigee Documentation - SpikeArrest policy
apigee Documentation - Quota policy
SAP Help - Flows
SAP Help - Condition Strings
SAP Help - Policy Types
