# TestAPIRepository
TestAPIRepository Umesh

Kyma ❌
Cloud Foundry ✅
Important - This part of the tutorial is required for Cloud Foundry deployments only!

As your SaaS application contains an API that allows the SaaS consumers to interact programmatically with their tenant database containers, you need to ensure that your API endpoints are properly managed and monitored. For this purpose, you should implement features like rate limiting to prevent e.g., DoS attacks. Furthermore, you can ensure fair usage of the resources among your consumers by e.g., setting up a quota depending on the chosen plan. A premium consumer might be eligible to send more requests per second than a standard consumer. Proper monitoring of your API will help you to analyze performance issues and to identify problems of your consumers.
