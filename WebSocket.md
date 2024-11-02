Let's go through creating the solution step-by-step, including setting up the project, configuring `launchSettings.json`, and creating the required classes and an `index.html` for testing the WebSocket. This guide assumes you're using .NET CLI and Visual Studio Code or Visual Studio, but you can adapt it for other environments.

### Step 1: Create the Solution and Project

1. **Create the Solution**:
   ```bash
   dotnet new sln -n WebSocketSolution
   cd WebSocketSolution
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

1. In the root of the `WebSocketApp` project, create a folder named `wwwroot`.
2. Inside `wwwroot`, create an `index.html` file.
3. Add the following HTML and JavaScript code to `index.html` to test the WebSocket connection:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Echo Test</title>
</head>
<body>
    <h1>WebSocket Echo Test</h1>
    <input type="text" id="messageInput" placeholder="Enter a message" />
    <button onclick="sendMessage()">Send</button>
    <ul id="messages"></ul>

    <script>
        const socket = new WebSocket("ws://localhost:5000/ws");

        socket.onopen = () => {
            console.log("Connected to WebSocket");
        };

        socket.onmessage = (event) => {
            const messagesList = document.getElementById("messages");
            const newMessage = document.createElement("li");
            newMessage.textContent = `Received: ${event.data}`;
            messagesList.appendChild(newMessage);
        };

        socket.onclose = () => {
            console.log("WebSocket connection closed");
        };

        function sendMessage() {
            const input = document.getElementById("messageInput");
            socket.send(input.value);
            input.value = "";
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

2. Open `http://localhost:5000/index.html` in your browser. You should see the WebSocket Echo Test interface. Enter a message, click **Send**, and it should echo the message back, appearing in the list below.

This completes the setup for a WebSocket-based echo server with a simple client interface to test the connection!
