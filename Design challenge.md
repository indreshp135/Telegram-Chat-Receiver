
> **Indresh** - _(03/10/2023 18:33:34)_
```
Hi boys
```
---
    
> **Indresh** - _(03/10/2023 18:41:05)_
```
In .NET Framework, you typically don't have a Startup class as you would in ASP.NET Core for configuring your application. Instead, you need to manually set up your test environment for the HTTP server. Here's an example of how to set up a test environment for an HTTP server without a Startup class:

csharp
using NUnit.Framework;
using System.Net;
using System.Net.Http;
using System.Web.Http;
using System.Web.Http.SelfHost;
using YourNamespace; // Replace with your actual namespace

namespace YourNamespace.Tests // Create a separate test namespace
{
    [TestFixture]
    public class ServerControllerTests
    {
        private HttpSelfHostServer _server;
        private HttpClient _httpClient;

        [SetUp]
        public void Setup()
        {
            // Configure the HttpSelfHostConfiguration for your Web API
            var config = new HttpSelfHostConfiguration("http://localhost:8080"); // Adjust the base address
            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );

            // Create the HttpSelfHostServer
            _server = new HttpSelfHostServer(config);

            // Create an HttpClient configured to send requests to the hosted application
            _httpClient = new HttpClient(_server);
        }

        [TearDown]
        public void TearDown()
        {
            // Dispose of the HttpSelfHostServer and HttpClient
            _server.CloseAsync().Wait();
            _server.Dispose();
            _httpClient.Dispose();
        }

        [Test]
        public async Task Ping_Returns200AndPong()
        {
            // Arrange
            var expectedResponse = "pong";

            // Act
            var response = await _httpClient.GetAsync("/api/ping");

            // Assert
            Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
            var responseContent = await response.Content.ReadAsStringAsync();
            Assert.AreEqual(expectedResponse, responseContent);
        }
    }
}

In this example:

1. We configure the HttpSelfHostConfiguration with the appropriate base address and route setup for your Web API.

2. We create the HttpSelfHostServer based on the configuration.

3. We create an HttpClient configured to send requests to the hosted application.

4. In the TearDown method, we close and dispose of the HttpSelfHostServer and the HttpClient to clean up resources.

Ensure that you adjust the base address, route template, and other configuration settings according to your actual Web API setup.
```
---
    