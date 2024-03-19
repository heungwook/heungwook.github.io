---
title: 02. IIS vs Kestrel for HTTPS and WSS
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: doc202401_02_iis_vs_kestrel.html
folder: doc202401
---

## IIS vs Kestrel for HTTPS and WSS

### IIS and Backend Services

Initially, I configured the SSL WebSocket service through IIS, because IIS does everything for web connections including the SSL certificate and HTTPS connections. So there was nothing to do on the application side for SSL and HTTPS. However, it did not work as I tested in the development environment using Kestrel. With IIS, the backend TCP socket server was very unstable, and I couldn't determine the cause. I suspect that the threading of the IIS worker process might have affected it.

### Kestrel with SSL Certificate

Until previous project, all my ASP.NET apps were running on Windows, so IIS had handled all HTTP/HTTPS connections. I've never configured SSL for an ASP.NET app with Kestrel. Here are the steps to configure Kestrel SSL services.

- Development Environment

    - [Create Self-signed certificate for localhost SSL](https://learn.microsoft.com/en-us/dotnet/core/additional-tools/self-signed-certificates-guide#create-a-self-signed-certificate)
        > dotnet dev-certs https -ep $env:USERPROFILE\.aspnet\https\localhost.pfx -p PASSWD<br/>
        > dotnet dev-certs https --trust
    - [Kestrel endpoint configuration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/endpoints?view=aspnetcore-8.0) for development/debug.
        
        ```CSharp
        public static void Main(string[] args)
        {
            int httpsPort = ServerCfg.WebSocketPort;
            int httpPort = ServerCfg.OrionWebSocketPort;

            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.ConfigureKestrel(serverOptions =>
                    {
                        serverOptions.Listen(IPAddress.Any, httpPort);
                        serverOptions.ListenAnyIP(httpsPort, listenOptions =>
                        {
                            listenOptions.UseHttps(httpsOptions =>
                            {
                                var localhostCert = CertificateLoader.LoadFromStoreCert(
                                    "localhost", "My", StoreLocation.CurrentUser,
                                    allowInvalid: true);
                                var certs = new Dictionary<string, X509Certificate2>(StringComparer.OrdinalIgnoreCase)
                                    {
                                        { "localhost", localhostCert },
                                    };

                                httpsOptions.ServerCertificateSelector = (connectionContext, name) =>
                                {
                                    if (name != null && certs.TryGetValue(name, out var cert))
                                    {
                                        return cert;
                                    }

                                    return localhostCert;
                                };
                            });
                        });

                    });
                    webBuilder.UseStartup<Startup>();
                })
                .Build()
                .Run();
        }
        ```


- Production Configuration

    - Purchase an SSL certificate and convert it to a .pfx file.

    - Configure SSL (assuming the certificate's .pfx file is located in the same folder as the execution assembly).
    
        ```CSharp
        ...
        serverOptions.Listen(IPAddress.Any, httpPort);
        serverOptions.ListenAnyIP(httpsPort, listenOptions =>
        {
            listenOptions.UseHttps(httpsOptions =>
            {
                string curAssemblyPath = new System.Uri(System.Reflection.Assembly.GetExecutingAssembly().Location).LocalPath;
                string assemblyFolder = Path.GetDirectoryName(curAssemblyPath);
                string certPath = Path.Combine(assemblyFolder, "webedit01.oryonsoft.com.pfx");
                listenOptions.UseHttps(certPath, "PASSWD");
            });
        });
        ...
        ```
