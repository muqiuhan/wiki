#csharp #dotnet 

In the early days of computing, software was pretty much locked into specific hardware setups. Then came operating systems, providing a layer of abstraction over the hardware. However, that still didn’t quite cut it. Businesses wanted to write their apps once and have them work on all sorts of platforms. This demand gave birth to higher-level languages that brought along their own runtimes, making it possible for software to run seamlessly on different operating systems.

Today, many organizations manage a multitude of programming languages for distinct tasks. While several runtimes, such as .NET’s Common Language Runtime (“CLR”) and Oracle’s Java Virtual Machine (“JVM”), support multiple languages, they still operate within isolated ecosystems. Each ecosystem has its package managers and libraries, often leading to unnecessary code duplication and duplicated efforts.

WebAssembly (Wasm) to the rescue! Wasm allows developers to create applications and libraries in their language of choice, meaning you can build a library in one language and smoothly integrate it with others.

Microsoft has long recognized Wasm’s potential and has heavily invested in [Blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor). Blazor runs .NET code in the browser. This way, C# and F# programmers can reuse their existing skills to write interactive apps for the web. In .NET 8, Microsoft is adding experimental support for WASI too. Which means .NET apps compiled to Wasm can now run on the server, computers, and mobile phones!

That’s where Extism steps in. Languages have varying levels of support for running and compiling to Wasm, and capabilities may change depending on what runtime is being used. Extism ensures that your plugins will run consistently across languages and platforms. We’ve [previously talked about the added value of Extism](https://dylibso.com/blog/why-extism/), but in a nutshell, Extism offers a set of unified and user-friendly Plugin Development Kits (PDKs) and Software Development Kits (SDKs) that make it easy for you to develop and run plugins in your preferred programming language.

## Writing a sample plugin
Extism makes it all the more convenient to build Wasm plugins. For example, consider this scenario: you have a messaging bot platform, and you want to empower your users to create their own bots. Typically, messaging platforms provide webhooks that users can use to write bots. But we want to invert the situation: we run the users code on our own infrastructure! Here’s a glimpse of how it could look when you use the [.NET Extism PDK](https://github.com/extism/dotnet-pdk):

```C#
  static class Plugin
{
    [UnmanagedCallersOnly(EntryPoint = "bot_name")]
    public static void BotName()
    {
        Pdk.SetOutput("weather bot");
    }

    [UnmanagedCallersOnly(EntryPoint = "respond")]
    public static void Respond()
    {
        var message = Pdk.GetInputString();

        if (message.Contains("hi", StringComparison.OrdinalIgnoreCase))
        {
            Reply("Hello :-)");
        }
        else if (message.Contains("weather", StringComparison.OrdinalIgnoreCase))
        {
            // Get secrets and configuration from the Host
            if (!Pdk.TryGetConfig("weather-api-key", out var apiKey))
            {
                throw new Exception("Beep boop malfunction detected: API Key is not configured!");
            }

            var block = MemoryBlock.Find(Env.GetUserInfo());
            var json = block.ReadString();
            var userInfo = JsonSerializer.Deserialize<UserInfo>(json);

            // Call HTTP APIs
            var query = $"https://api.weatherapi.com/v1/current.json?key={apiKey}&q={userInfo.City}&aqi=no";
            var response = Pdk.SendRequest(new HttpRequest(query));
            var responseJson = response.Body.ReadString();
            var apiResponse = JsonSerializer.Deserialize<ApiResponse>(responseJson);

            Reply($"The current temparature in {userInfo.City} is {apiResponse.current.temp_c}°C");
        }
        else
        {
            Reply("""
                Hi, I am the weather bot. Commands:
                1. Hi
                2. How's the weather?
                """);
        }
    }

    static void Reply(string message)
    {
        var block = Pdk.Allocate(message);
        Env.SendMessage(block.Offset);
    }
}

// Import functions from the host
static class Env
{
    [DllImport("extism", EntryPoint = "send_message")]
    public static extern void SendMessage(ulong offset);

    [DllImport("extism", EntryPoint = "user_info")]
    public static extern ulong GetUserInfo();
}
```

This small example demonstrates how we can easily import functions from the host, share data between the host and the plugin, and make HTTP requests. The host has full control over which HTTP hosts the plugin can make requests to and which files the plugin has access to.

We have also exported two functions for the host: `bot_name` provides the name of the bot and `respond` can process a message sent by the user. The .NET Extism PDK add the following capabilities on top of the .NET WASI SDK:
1. Support for `DllImport` for importing functions from the host.
2. Support for `UnmanagedCallersOnly` for exporting C# and F# functions.
3. A global exception handler that propagates the exceptions back to the host.
4. Easily share data between the plugin and the host.
Extism has PDKs for [many popular programming languages](https://extism.org/docs/category/write-a-plug-in), so your users can write plugins in their favorite programming language and it would still work the same way!

## Running plugins from the Host
Running plugins is equally easy thanks to our [Host SDKs](https://extism.org/docs/category/integrate-into-your-codebase), here is how you’d call the plugin above using our [.NET SDK](https://github.com/extism/dotnet-sdk):
```C#
var manifest = new Manifest(new PathWasmSource("Plugin.wasm"))
{
	// Provide configurations and secrets for the plugins
	Config = new Dictionary<string, string>
	{
		{ "weather-api-key", Environment.GetEnvironmentVariable("weather-api-key") }
	},

 	// Control which HTTP hosts the plugins can call
 	AllowedHosts = ["api.weatherapi.com"]
};

// Provide custom host functions
var functions = new[]
{
    HostFunction.FromMethod("send_message", IntPtr.Zero, (CurrentPlugin plugin, long offset) =>
    {
        var message = plugin.ReadString(offset);
        Console.WriteLine($"bot says: {message}");
    }),

    HostFunction.FromMethod("user_info", IntPtr.Zero, (CurrentPlugin plugin) =>
    {
        var json = JsonSerializer.Serialize(new UserInfo
        {
            FullName = "John Smith",
            City = "New York"
        });

        return plugin.WriteString(json);
    })
};

var plugin = new Plugin(manifest, functions, withWasi: true);

// Call functions exported by the plugin
var botName = plugin.Call("bot_name", "");
Console.WriteLine($"Bot Name: {botName}");

// Call respond with an empty input
plugin.Call("respond", "");

while (true)
{
    Console.Write("> ");
    var message = Console.ReadLine();

    // Easily cancel plugin calls
    var cts = new CancellationTokenSource();
    cts.CancelAfter(TimeSpan.FromSeconds(1));

    plugin.Call("respond", message, cts.Token);
}
```

And here is the result of running the host:

```
PS D:\x\dotnet\isomophic-dotnet-demo\Host> dotnet run
Bot Name: weather bot
bot says: Hi, I am the weather bot. Commands:
1. Hi
2. How's the weather?
> hi
bot says: Hello :-)
> weather?
bot says: The current temparature in New York is 10.6°C
>
```
While this example is very simple, it serves as the foundation for a robust and highly extensible bot framework. Utilizing Wasm for this scenario offers a host of advantages:

1. **Sandboxed Plugins:** Wasm ensures that plugins operate within a secure sandbox. They can only access resources such as memory, files, and sockets if explicitly allowed by the Host.
2. **Language Flexibility:** Your users can develop plugins in their preferred programming language.
3. **Resource Management:** You can set precise limits on memory usage and execution timeouts, enhancing control over your bot framework.
4. **Less network traffic:** Since you’re running the plugins on your own server, there is no need for webhooks which can improve the responsiveness of the bots.
The complete code for this sample is [available on GitHub](https://github.com/dylibso/isomorphic-dotnet-demo).

Finally, we want to thank the incredible .NET team for their invaluable support and guidance through the WASI SDK challenges. We’re excited about the future of .NET in Wasm.

[Muhammad Azeez](https://twitter.com/mhmd_azeez), Senior Software Engineer