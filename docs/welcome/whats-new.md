# What's New in Steeltoe 3.0

## New Features and Changes

* We made the Steeltoe libraries more platform agnostic to allow for better extensibility into other platforms. So, we took the initial step of creating abstractions for our core components.  This will allow future extensibility for our libraries and grow the community into other areas.
* We added and renamed many of the packages to support our new features and to provide a consistent package naming.
* We redirected our focus to support only .NET Core.  This decision was based on the direction the Microsoft .NET team is taking the project and to support our users moving forward. We will still be supporting .NET Framework in our 2.x release line.
* We have added some great new features and here are some of the highlights:
   * Automatic wiring and configuration of Messaging APIs with RabbitMQ
   * Kubernetes support for configuration (ConfigMap and Secrets) and service discovery using Kubernetes .NET Client.
   * Added Health Groups for `readiness` and `liveness` endpoints which are grouped under the `/health` endpoint
   * Metrics now uses `EventSource` and `EventCounter`, along with a new prometheus exporter that now uses OpenTelemetry metrics packages
   * Distributed tracing library now has new exporters and updated internal libraries from OpenCensus to OpenTelemetry
   * Pluggable architecture for Service Discovery (Consul, Eureka, and Kubernetes
   * New Connector for CosmosDB
   * The `/heapdump` actuator endpoint now supports heap dumps on Linux
   * Circuit Breaker using Hystrix now using the Prometheus endpoint for easier consumption of events on Prometheus supported services
   * Added mTLS support and service to service authentication using rotating certificates

   
 ## Package Name Changes
 | Steeltoe 2.x | Steeltoe 3.x |
 | ------------ | ------------ |
 |N/A|Steeltoe.CircuitBreaker.Abstractions|
 | Steeltoe.CircuitBreaker.Hystrix.MetricsEventsCore | Same |
 | Steeltoe.CircuitBreaker.Hystrix.MetricsStreamAutofac | N/A |
 | Steeltoe.CircuitBreaker.Hystrix.MetricsStreamCore | Same |
 | Steeltoe.CircuitBreaker.HystrixAutofac | N/A |
 | Steeltoe.CircuitBreaker.HystrixBase  | Same |
 | Steeltoe.CircuitBreaker.HystrixCore  | Same |
 | Steeltoe.CloudFoundry.Connector.EF6Autofac | N/A|
 | Steeltoe.CloudFoundry.Connector.EF6Core  | N/A |
 | Steeltoe.CloudFoundry.Connector.EFCore  | N/A |
 | Steeltoe.CloudFoundry.ConnectorAutofac  | N/A |
 | Steeltoe.CloudFoundry.ConnectorBase  | N/A |
 | Steeltoe.CloudFoundry.ConnectorCore  | N/A |
 | Steeltoe.Common  | Same |
 |N/A| Steeltoe.Common.Abstractions |
 | Steeltoe.Common.Autofac  | N/A |
 | Steeltoe.Common.Hosting  | Same |
 | Steeltoe.Common.Http  | Same |
 |N/A|Steeltoe.Common.Kubernetes|
 | Steeltoe.Common.Net  | Same |
 |N/A| Steeltoe.Common.Retry |
 | Steeltoe.Common.Security  | Same |
 |N/A| Steeltoe.Connector.Abstractions 
 |N/A| Steeltoe.Connector.CloudFoundry 
 |N/A| Steeltoe.Connector.ConnectorBase
 |N/A| Steeltoe.Connector.ConnectorCore
 |N/A| Steeltoe.Connector.EF6Core
 |N/A| Steeltoe.Connector.EFCore
 |N/A| Steeltoe.Discovery.Abstractions
 |Steeltoe.Discovery.ClientAutofac | N/A|
 |N/A| Steeltoe.Discovery.ClientBase
 | Steeltoe.Discovery.ClientCore  | Same |
 | Steeltoe.Discovery.ConsulBase  | Steeltoe.Discovery.Consul |
 | Steeltoe.Discovery.EurekaBase  | Steeltoe.Discovery.Eureka |
 |N/A| Steeltoe.Discovery.Kubernetes|
 |N/A| Steeltoe.Extensions.Configuration.Abstractions|
 | Steeltoe.Extensions.Configuration.CloudFoundryAutofac |N/A|  
 | Steeltoe.Extensions.Configuration.CloudFoundryBase  | Same |
 | Steeltoe.Extensions.Configuration.CloudFoundryCore  | Same |
 | Steeltoe.Extensions.Configuration.ConfigServerAutofac  | N/A |
 | Steeltoe.Extensions.Configuration.ConfigServerBase  | Same |
 | Steeltoe.Extensions.Configuration.ConfigServerCore  | Same |
 |N/A| Steeltoe.Extensions.Configuration.KubernetesBase|
 |N/A| Steeltoe.Extensions.Configuration.KubernetesCore|
 | Steeltoe.Extensions.Configuration.PlaceholderBase  | Same |
 | Steeltoe.Extensions.Configuration.PlaceholderCore  | Same |
 | Steeltoe.Extensions.Configuration.RandomValueBase  | Same |
 |N/A| Steeltoe.Extensions.Logging.Abstractions |
 | Steeltoe.Extensions.Logging.DynamicLogger | Same |
 | Steeltoe.Extensions.Logging.SerilogDynamicLogger  |Steeltoe.Extensions.Logging.DynamicSerilogBase |
 | Steeltoe.Extensions.Logging.SerilogDynamicLogger | Steeltoe.Extensions.Logging.DynamicSerilogCore |
 |N/A| Steeltoe.Integration.Abstractions * |
 |N/A| Steeltoe.Integration.IntegrationBase * |
 |N/A| Steeltoe.Management.Abstractions |
 | Steeltoe.Management.CloudFoundryCore  | Same |
 | Steeltoe.Management.Diagnostics  | Same |
 | Steeltoe.Management.EndpointBase | Same | 
 | Steeltoe.Management.EndpointCore | Same | 
 | Steeltoe.Management.EndpointOwin |N/A| 
 | Steeltoe.Management.EndpointOwinAutofac  |N/A|
 | Steeltoe.Management.EndpointWeb  |N/A|
 | Steeltoe.Management.ExporterBase  |N/A|
 | Steeltoe.Management.ExporterCore  |N/A|
 |N/A| Steeltoe.Management.KubernetesCore |
 | Steeltoe.Management.OpenCensus  | N/A|
 | Steeltoe.Management.OpenCensus.Abstractions  |N/A|
 | Steeltoe.Management.OpenCensus.ZipkinExporter  |N/A|
 | Steeltoe.Management.OpenCensusBase  |N/A|
 |N/A| Steeltoe.Management.OpenTelemetryBase |
 | Steeltoe.Management.TaskCore  | Same |
 | Steeltoe.Management.TracingBase  | Same |
 | Steeltoe.Management.TracingCore  | Same |
 |N/A| Steeltoe.Messaging.Abstractions|
 |N/A| Steeltoe.Messaging.MessagingBase|
 |N/A| Steeltoe.Messaging.RabbitMQ|
 | Steeltoe.Security.Authentication.CloudFoundryBase  | Same |
 | Steeltoe.Security.Authentication.CloudFoundryCore  | Same |
 | Steeltoe.Security.Authentication.CloudFoundryOwin  |N/A|
 | Steeltoe.Security.Authentication.CloudFoundryWcf  |N/A|
 |N/A| Steeltoe.Security.Authentication.MtlsCore |
 | Steeltoe.Security.DataProtection.CredHubBase  | Same |
 | Steeltoe.Security.DataProtection.CredHubCore  | Same |
 | Steeltoe.Security.DataProtection.RedisCore  | Same |
 |N/A| Steeltoe.Stream.Abstractions * |
 |N/A| Steeltoe.Stream.StreamBase * |
 
  \* Experimental packages
 
 ## Release Notes
 Release notes for all releases can be found on the [Steeltoe releases](https://github.com/SteeltoeOSS/Steeltoe/releases) section on GitHub. 
 
 