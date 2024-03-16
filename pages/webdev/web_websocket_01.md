---
title: WebSocket (migration from WebAPI)
tags: [wpf]
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: web_websocket_01.html
folder: webdev
---

## Overview

For years, developing a new web-based user interface for Orion Report Editor is in progress. Orion itself has a feature to create HTML5-Canvas(Javascript) outputs, but these outputs are static HTML pages. Web Editor module originally (in 2017) developed with ASP.NET WebAPI (.NET Framework) and Javascript. In late 2023, it was redesigned with ASP.NET Core WebAPI(.NET 8) and TypeScript.

In March 2024, since I got the first order of Web Editor module, I'm integrating Orion's Web Editor module into customer's environment. And, WebAPI has been replaced with WebSocket.

- Web Editor needs to be embedded into the customer's shopping mall site which is hosted in the Cafe24 shopping mall services.
- Web Editor will be a part(Modal or Frame) of Cafe24's product page with IFRME or EMBED tags.
- SSL(HTTPS, WSS) is required.

****

- Orion Web Editor with WebAPI 
    ![Orion Web Editor with WebAPI](WebEdit_WebAPI.png)

*****

- Orion Web Editor with WebSocket 
    ![Orion Web Editor with WebSocket](WebEdit_WebSocket.png)

****




