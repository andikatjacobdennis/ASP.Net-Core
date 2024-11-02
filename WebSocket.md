# WebSocket

Let's go through creating the solution step-by-step, including setting up the project, configuring `launchSettings.json`, and creating the required classes and an `index.html` for testing the WebSocket. This guide assumes you're using .NET CLI and Visual Studio Code or Visual Studio, but you can adapt it for other environments.

### Step 1: Create the Solution and Project

1. **Create the Solution**:
   ```bash
   
   md WebSocketExample
   cd WebSocketExample
   dotnet new sln -n WebSocketExample
   
   ```

2. **Create the ASP.NET Core Web API Project**:
   ```bash
   dotnet new web -n WebSocketApp
   ```

3. **Add the Project to the Solution**:
   ```bash
   dotnet sln add WebSocketApp/WebSocketApp.csproj
   ```

4. **Navigate to the Project Directory**:
   ```bash
   cd WebSocketApp
   ```

### Step 2: Update `launchSettings.json`

The `launchSettings.json` file is in the `Properties` folder in the project directory (`WebSocketApp/Properties/launchSettings.json`). This file is used to configure the environment for running the application locally.

1. Open `launchSettings.json` in an editor.
2. Add the WebSocket path in the `launchUrl` property. It might look like this after editing:

```json
{
  "profiles": {
    "WebSocketApp": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "ws",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

### Step 3: Create the Abstract Class `WebSocketHub<T>`

1. In the `WebSocketApp` project folder, create a new folder named `WebSockets` for organization.
2. Inside the `WebSockets` folder, create a file named `WebSocketHub.cs`.
3. Add the following code to `WebSocketHub.cs`:

```csharp
using System.Net.WebSockets;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;

namespace WebSocketApp.WebSockets
{
    public abstract class WebSocketHub<T>
    {
        protected abstract Task OnConnectedAsync(WebSocket socket, T context);
        protected abstract Task OnDisconnectedAsync(WebSocket socket, T context);
        protected abstract Task OnMessageReceivedAsync(WebSocket socket, WebSocketReceiveResult result, byte[] buffer, T context);

        public async Task HandleWebSocketAsync(HttpContext httpContext, T context)
        {
            if (httpContext.WebSockets.IsWebSocketRequest)
            {
                var socket = await httpContext.WebSockets.AcceptWebSocketAsync();
                await OnConnectedAsync(socket, context);
                await ProcessWebSocketMessages(socket, context);
                await OnDisconnectedAsync(socket, context);
            }
            else
            {
                httpContext.Response.StatusCode = 400;
            }
        }

        private async Task ProcessWebSocketMessages(WebSocket socket, T context)
        {
            var buffer = new byte[1024 * 4];
            WebSocketReceiveResult result = null;

            try
            {
                do
                {
                    result = await socket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
                    if (result.MessageType == WebSocketMessageType.Close)
                        break;

                    await OnMessageReceivedAsync(socket, result, buffer, context);
                }
                while (!result.CloseStatus.HasValue);
            }
            finally
            {
                await socket.CloseAsync(
                    result?.CloseStatus ?? WebSocketCloseStatus.NormalClosure,
                    result?.CloseStatusDescription,
                    CancellationToken.None
                );
            }
        }
    }
}
```

### Step 4: Create the `EchoWebSocketHub` Class

1. In the `WebSockets` folder, create a new file named `EchoWebSocketHub.cs`.
2. Add the following code to `EchoWebSocketHub.cs`:

```csharp
using System.Net.WebSockets;
using System.Threading;
using System.Threading.Tasks;

namespace WebSocketApp.WebSockets
{
    public class EchoWebSocketHub : WebSocketHub<string> // Example using string as T
    {
        protected override Task OnConnectedAsync(WebSocket socket, string context)
        {
            // Logic for when a connection is established
            return Task.CompletedTask;
        }

        protected override Task OnDisconnectedAsync(WebSocket socket, string context)
        {
            // Logic for when a connection is closed
            return Task.CompletedTask;
        }

        protected override async Task OnMessageReceivedAsync(WebSocket socket, WebSocketReceiveResult result, byte[] buffer, string context)
        {
            // Echo the received message back to the client
            await socket.SendAsync(new ArraySegment<byte>(buffer, 0, result.Count), result.MessageType, result.EndOfMessage, CancellationToken.None);
        }
    }
}
```

### Step 5: Update `Program.cs` to Use WebSocket Middleware

1. Open `Program.cs` in the root of the `WebSocketApp` project.
2. Replace the contents with the following code:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using WebSocketApp.WebSockets;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

var app = builder.Build();

app.UseWebSockets();
app.UseCors();

var echoWebSocketHub = new EchoWebSocketHub();

app.Map("/ws", async context =>
{
    var connectionContext = "ExampleContext"; // Replace with actual context data as needed
    await echoWebSocketHub.HandleWebSocketAsync(context, connectionContext);
});

app.Run();
```

### Step 6: Create `index.html` to Test WebSocket

1. In the root of the `WebSocketApp` project
2. Add the following HTML and JavaScript code to `index.html` to test the WebSocket connection:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Client</title>
</head>
<body>
    <h1>WebSocket Client</h1>
    <button onclick="connect()">Connect to WebSocket</button>
    <button onclick="sendMessage()">Send Message</button>
    <button onclick="disconnect()">Disconnect</button>
    <div id="output"></div>

    <script>
        let socket;

        function connect() {
            try {
                socket = new WebSocket("ws://localhost:5000/ws");

                socket.onopen = () => {
                    log("WebSocket connection established");
                    setTimeout(() => socket.send("Hello, server!"), 1000); // Delay before first message
                };

                socket.onmessage = (event) => {
                    log("Message from server: " + event.data);
                };

                socket.onclose = (event) => {
                    log("WebSocket connection closed: " + event.reason);
                };

                socket.onerror = (error) => {
                    log("WebSocket error: " + error.message);
                };
            } catch (error) {
                log("WebSocket connection failed: " + error.message);
            }
        }

        function sendMessage() {
            if (socket && socket.readyState === WebSocket.OPEN) {
                const message = "Hello, server!";
                socket.send(message);
                log("Message sent to server: " + message);
            } else {
                log("WebSocket is not connected.");
            }
        }

        function disconnect() {
            if (socket) {
                socket.close();
            }
        }

        function log(message) {
            const output = document.getElementById("output");
            output.innerHTML += `<p>${message}</p>`;
        }
    </script>
</body>
</html>
```

### Step 7: Run the Application

1. Run the application from the command line:
   ```bash
   dotnet run
   ```

2. Open `index.html` in your browser. You should see the WebSocket Echo Test interface. Enter a message, click **Send**, and it should echo the message back, appearing in the list below.

This completes the setup for a WebSocket-based echo server with a simple client interface to test the connection!
