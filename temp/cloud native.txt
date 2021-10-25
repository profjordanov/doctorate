Introduction to cloud-native applications 1
Cloud-native computing ..3
Defining cloud native .4
The cloud... 5
Modern design.6
Microservices 8
Containers ... 10
Backing services 13
Automation. 14
Candidate apps for cloud native. 16
Summary.. 18
Introducing eShopOnContainers reference app  20
Features and requirements ... 21
Overview of the code .. 23
Understanding microservices... 25
Mapping eShopOnContainers to Azure Services. 25
Container orchestration and clustering... 26
API Gateway ... 26
Data ... 27
Event Bus . 28
Resiliency. 28
Deploying eShopOnContainers to Azure 28
Azure Kubernetes Service . 28
Deploying to Azure Kubernetes Service using Helm . 28
Azure Dev Spaces. 30
Azure Functions and Logic Apps (Serverless) ... 32
Centralized configuration .. 32
Azure App Configuration .. 32

ii Contents
Azure Key Vault. 33
Configuration in eShop.. 33
References... 34
Scaling cloud-native applications 35
Leveraging containers and orchestrators 35
Challenges with monolithic deployments .. 35
What are the benefits of containers and orchestrators? .. 37
What are the scaling benefits? 39
What scenarios are ideal for containers and orchestrators?... 41
When should you avoid using containers and orchestrators?... 41
Development resources. 41
Leveraging serverless functions .. 46
What is serverless?... 46
What challenges are solved by serverless?  47
What is the difference between a microservice and a serverless function? . 47
What scenarios are appropriate for serverless?... 47
When should you avoid serverless?.. 48
Combining containers and serverless approaches.. 49
When does it make sense to use containers with serverless? 49
When should you avoid using containers with Azure Functions?  49
How to combine serverless and Docker containers ... 49
How to combine serverless and Kubernetes with KEDA .. 50
Deploying containers in Azure  50
Azure Container Registry .. 50
ACR Tasks  52
Azure Kubernetes Service . 52
Azure Dev Spaces. 53
Scaling containers and serverless applications . 54
The simple solution: scaling up .. 54
Scaling out cloud-native apps  55
Other container deployment options ... 56
When does it make sense to deploy to App Service for Containers? . 56

iii Contents
How to deploy to App Service for Containers.. 56
When does it make sense to deploy to Azure Container Instances? .. 56
How to deploy an app to Azure Container Instances 56
References... 57
Cloud-native communication patterns ... 59
Communication considerations .. 59
Front-end client communication 61
Ocelot Gateway. 63
Azure Application Gateway.. 64
Azure API Management. 65
Real-time communication  68
Service-to-service communication  69
Queries . 70
Commands.. 73
Events 76
gRPC... 82
What is gRPC? ... 82
gRPC Benefits  82
Protocol Buffers  83
gRPC support in .NET . 83
gRPC usage. 84
gRPC implementation  85
Looking ahead... 86
Service Mesh communication infrastructure . 87
Summary.. 88
Distributed data. 90
Database-per-microservice, why? .. 91
Cross-service queries... 92
Distributed transactions . 93
High volume data . 95
CQRS . 95
Event sourcing... 96

iv Contents
Relational vs. NoSQL data . 98
The CAP theorem . 99
Considerations for relational vs. NoSQL systems..101
Database as a Service ...101
Azure relational databases .102
Azure SQL Database..102
Open-source databases in Azure.103
NoSQL data in Azure 104
NewSQL databases108
Data migration to the cloud ..109
Caching in a cloud-native app...110
Why?110
Caching architecture.110
Azure Cache for Redis ..111
Elasticsearch in a cloud-native app .112
Summary113
Cloud-native resiliency ... 115
Application resiliency patterns ..116
Circuit breaker pattern.118
Testing for resiliency.119
Azure platform resiliency .119
Design with resiliency...119
Design with redundancy..120
Design for scalability.122
Built-in retry in services ...123
Resilient communications124
Service mesh 124
Istio and Envoy126
Integration with Azure Kubernetes Services ...126
Monitoring and health 128
Observability patterns...128
When to use logging128

v Contents
Challenges with detecting and responding to potential app health issues ...132
Challenges with reacting to critical problems in cloud-native apps..132
Logging with Elastic Stack...133
Elastic Stack ..133
What are the advantages of Elastic Stack? ..134
Logstash.134
Elastic search135
Visualizing information with Kibana web dashboards 135
Installing Elastic Stack on Azure...136
References.136
Monitoring in Azure Kubernetes Services.136
Azure Monitor for Containers ...136
Log.Finalize() 138
Azure Monitor..138
Gathering logs and metrics139
Reporting data 139
Dashboards...140
Alerts ...142
References.143
Identity . 144
References .144
Authentication and authorization in cloud-native apps .144
References.145
Azure Active Directory ..145
References.145
IdentityServer for cloud-native applications146
Common web app scenarios .146
Getting started 147
Configuration...147
JavaScript clients 148
References.148
Security. 149

vi Contents
Azure security for cloud-native apps ..149
Threat modeling .150
Principle of least privilege...150
Penetration testing151
Monitoring151
Securing the build..151
Building secure code 152
Built-in security ...152
Azure network infrastructure.152
Role-based access control for restricting access to Azure resources154
Security Principals ..154
Roles155
Scopes 156
Deny 156
Checking access..156
Securing secrets..157
Azure Key Vault...157
Kubernetes157
Encryption in transit and at rest...158
Keeping secure162
DevOps . 163
Azure DevOps...164
GitHub Actions.165
Source control..165
Repository per microservice ..166
Single repository 168
Standard directory structure..169
Task management ..169
CI/CD pipelines 171
Azure Builds..172
Azure DevOps releases 174
Everybody gets a build pipeline...175

vii Contents
Versioning releases175
Feature flags .175
Implementing feature flags176
Infrastructure as code ...177
Azure Resource Manager templates ..177
Terraform...178
Azure CLI Scripts and Tasks179
Cloud Native Application Bundles ...180
DevOps Decisions ..182
References.182
Summary .. 183
