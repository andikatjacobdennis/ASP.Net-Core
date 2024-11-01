To create a WebSocket hub with an abstract class `WebSocketHub<T>`, we can implement a pattern similar to ASP.NET Core SignalR. This approach encapsulates common WebSocket handling logic and makes it easy to extend for specific types of messages. We’ll also create a basic client example to test the WebSocket server.

### Server-Side: WebSocket Hub Implementation

1. **Create an abstract `WebSocketHub<T>` class**.
2. **Create a concrete hub class** that inherits from `WebSocketHub<T>` and defines how to handle specific messages.
3. **Use the hub** in the WebSocket route.

Here’s how to do it:

#### Step 1: Define `WebSocketHub<T>`

```csharp
using System.Net.WebSockets;
using System.Text;
using System.Text.Json;

public abstract class WebSocketHub<T>
{
    public async Task HandleConnectionAsync(WebSocket webSocket)
    {
        var buffer = new byte[1024 * 4];
        
        while (webSocket.State == WebSocketState.Open)
        {
            var result = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            
            if (result.MessageType == WebSocketMessageType.Close)
            {
                await webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
            }
            else if (result.MessageType == WebSocketMessageType.Text)
            {
                var jsonMessage = Encoding.UTF8.GetString(buffer, 0, result.Count);
                var message = JsonSerializer.Deserialize<T>(jsonMessage);
                
                if (message != null)
                {
                    await OnMessageReceivedAsync(webSocket, message);
                }
            }
        }
    }

    protected abstract Task OnMessageReceivedAsync(WebSocket webSocket, T message);

    protected async Task SendMessageAsync(WebSocket webSocket, T message)
    {
        var jsonMessage = JsonSerializer.Serialize(message);
        var messageBuffer = Encoding.UTF8.GetBytes(jsonMessage);
        await webSocket.SendAsync(new ArraySegment<byte>(messageBuffer), WebSocketMessageType.Text, true, CancellationToken.None);
    }
}
```

- **`HandleConnectionAsync`**: Manages the WebSocket connection and deserializes incoming messages as type `T`.
- **`OnMessageReceivedAsync`**: An abstract method that will handle messages of type `T` (implemented in subclasses).
- **`SendMessageAsync`**: Sends messages of type `T` back to the client.

#### Step 2: Implement a Concrete Hub

Here’s an example hub that handles chat messages:

```csharp
public class ChatMessage
{
    public string User { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
}

public class ChatHub : WebSocketHub<ChatMessage>
{
    protected override async Task OnMessageReceivedAsync(WebSocket webSocket, ChatMessage message)
    {
        Console.WriteLine($"{message.User}: {message.Message}");
        
        // Echo the message back to the client
        await SendMessageAsync(webSocket, new ChatMessage
        {
            User = "Server",
            Message = $"Echo: {message.Message}"
        });
    }
}
```

#### Step 3: Configure WebSocket in `Program.cs`

Add the `ChatHub` to the WebSocket route in `Program.cs`:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseWebSockets();

app.Map("/ws", async context =>
{
    if (context.WebSockets.IsWebSocketRequest)
    {
        var chatHub = new ChatHub();
        using var webSocket = await context.WebSockets.AcceptWebSocketAsync();
        await chatHub.HandleConnectionAsync(webSocket);
    }
    else
    {
        context.Response.StatusCode = StatusCodes.Status400BadRequest;
    }
});

app.Run();
```

### Client-Side Code

Below is an example client written in JavaScript. You can run it in the browser console.

```javascript
const socket = new WebSocket("ws://localhost:5000/ws");

socket.onopen = () => {
    console.log("Connected to WebSocket server");
    
    // Send a test message
    const message = { user: "Client", message: "Hello, server!" };
    socket.send(JSON.stringify(message));
};

socket.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log("Received from server:", message);
};

socket.onclose = () => {
    console.log("WebSocket connection closed");
};

socket.onerror = (error) => {
    console.error("WebSocket error:", error);
};
```

### Explanation of Client Code

- **`socket.send(JSON.stringify(message))`**: Sends a message as a JSON string.
- **`socket.onmessage`**: Logs incoming messages from the server.
- **`socket.onclose`**: Notifies when the WebSocket connection closes.

This setup provides a reusable WebSocket handler with `WebSocketHub<T>`, allowing you to handle different types of messages by implementing new hubs based on the abstract class. The client-side code connects to the server, sends a message, and receives the echo response.
