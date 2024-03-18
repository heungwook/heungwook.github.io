---
title: 04. WebSocket Server in .NET 8
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: doc202401_04_websocket_server.html
folder: doc202401
---

## WebSocket Server in .NET 8

There is a well-designed [WebSocket Server/Client Sample](https://github.com/MV10/WebSocketExample) in .NET, and I modified it for the Orion WebSocket Server application.

As I mentioned earlier in this post, the Orion WebSocket Server is running on the Kestrel Web Server, and there are two services running: the WebSocket server service and the TCP Orion message server service.

- Startup Class

    ```CSharp
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            // register the background process to periodically check for new Orion server connections
            services.AddHostedService<OrionNetService>();
            // register our custom middleware since we use the IMiddleware factory approach
            services.AddTransient<WebSocketMiddleware>();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // enable websocket support
            app.UseWebSockets(new WebSocketOptions
            { 
                KeepAliveInterval = TimeSpan.FromSeconds(120),
                //ReceiveBufferSize = 4 * 1024
            });
            // add our custom middleware to the pipeline
            app.UseMiddleware<WebSocketMiddleware>();
        }
    }
}
    ```

### WebSocket Server




### TCP Orion Message Server


