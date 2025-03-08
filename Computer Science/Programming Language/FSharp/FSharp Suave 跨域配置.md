#fsharp #web #fp

- [CORS response with Suave](https://www.fssnip.net/mL/title/CORS-response-with-Suave)
- [Allow multiple headers with CORS in Suave](https://stackoverflow.com/questions/44359375/allow-multiple-headers-with-cors-in-suave)
```fsharp
open Suave
open Suave.Writers
open Suave.Filters
open Suave.Operators
  
let CORS =
    addHeader "Access-Control-Allow-Origin" "*"
    >=> setHeader "Access-Control-Allow-Headers" "token"
    >=> addHeader "Access-Control-Allow-Headers" "Content-Type"
    >=> addHeader "Access-Control-Allow-Methods" "GET"
    
let Apps: WebPart =
    GET
    >=> fun context ->
      context
      |> (Server.CORS
          >=> choose
            [
              (* 需要跨域的 WebPart 放在这里 *) 
            ])

startWebServer defaultConfig Apps
```