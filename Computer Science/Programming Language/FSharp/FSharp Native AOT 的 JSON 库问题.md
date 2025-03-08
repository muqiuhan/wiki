#fsharp #dotnet

- FSharp.SystemTextJson 不行
- FSharp.Json 不行
- __用 FSharp.Data 的 JsonProvider__ 
- 然后在 `PropertyGroup` 中设置 `<JsonSerializerIsReflectionEnabledByDefault> true </JsonSerializerIsReflectionEnabledByDefault>`