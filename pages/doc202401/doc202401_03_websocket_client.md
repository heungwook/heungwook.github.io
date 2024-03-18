---
title: 03. WebSocket Client in TypeScript
tags: []
keywords:
summary: 
sidebar: mydoc_sidebar
permalink: doc202401_03_webwocket_client.html
folder: doc202401
---


### Reconnecting WebSocket Client Library

ASP.NET Core supports [WebSocket](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.websockets?view=aspnetcore-8.0) and there is a TypeScript library([Reconnecting WebSocket](https://github.com/pladaria/reconnecting-websocket)) featuring automatic reconnection when the connection is lost.  

### Send Data Timeout and Callback

There are two types of messages: one-way and round-trip. Every message sent to the Orion Report Server receives a response to confirm that the message was processed by the Orion. The Message Handler class keeps track of each message, including handling message timeouts. 

- MessageHandler Class

    ```TypeScript
    export class MessageHandler {
        reconnWebSock: ReconnectingWebSocket;
        msgID: number;
        resultCallback: (result: ApiResult) => void | null;
        timeSpan: TimeSpan;
        timeoutMS: number;
        timeoutID: any;
        retryCount: number;
        retryMax: number;
        data: string;
        timeoutCallback: (event: MessageEvent) => void | null;
        result: ApiResult | null;
        success: boolean;
        isDisposed: boolean;

        constructor(reconnWebSock: ReconnectingWebSocket, msgID: number, resultCallback: (result: ApiResult) => void,
            timeoutMS: number, data: string, timeoutCallback: (event: MessageEvent) => void | null) {
            this.reconnWebSock = reconnWebSock;
            this.msgID = msgID;
            this.resultCallback = resultCallback;
            this.timeSpan = TimeSpan.zero;
            this.timeoutMS = timeoutMS; // 0 = no timeout
            this.retryCount = 0;
            this.retryMax = 3;
            this.data = data;
            this.timeoutCallback = timeoutCallback;
            this.result = null;
            this.success = false;
            this.isDisposed = false
            //
            this.dispose = this.dispose.bind(this);
            this.timerTick = this.timerTick.bind(this);
            //
            this.timeoutID = null;
            if (this.timeoutMS > 0) {
                this.timeoutID = setTimeout(this.timerTick, this.timeoutMS);
            }

        }

        dispose() {
            this.isDisposed = true;
            if (this.timeoutID !== null) {
                clearTimeout(this.timeoutID);
            }
        }

        timerTick() {
            this.retryCount++;
            if (this.retryCount < this.retryMax) {
                this.timeoutID = setTimeout(this.timerTick, this.timeoutMS);
            } else {
                if (this.timeoutCallback) {
                    const apiResult: ApiResult = new ApiResult();
                    apiResult.msgID = this.msgID;
                    apiResult.success = false;
                    apiResult.message = `Sending msgID: ${this.msgID} exceeded retry count. data = ${this.data}`;
                    const msgEvent: MessageEvent = new MessageEvent('timeout', { data: JSON.stringify(apiResult) });
                    this.timeoutCallback(msgEvent);
                }
                this.dispose();
            }
        }
    }
    ```


- One-way message

    - No callback function is provided.
    - When the dummy response (the receipt for successful transfer) from Orion Server is not received within the timeout period, it records a timeout message to the console.

- Round-trip message

    - The sendData() function adds a new message handler to the message list and sends it to the Orion WebSocket Server.

        ```TypeScript
        ...
        async sendData(data: object, resultCallback: ((result: ApiResult) => void), timeoutMS: number = this.defaultTimeoutMS): Promise<void> {
            this.checkDisposedMessageHandlers();

            if (this.reconnWebSock?.readyState !== ReconnectingWebSocket.OPEN) {
                console.log("sendData() ERR : this.reconnWebSock?.readyState !== WebSocket.OPEN");
                return;
            }

            data = { '@MSGID': this.messageCount, ...data };
            const jsonData = JSON.stringify(data);
            const msgHandle = new MessageHandler(
                this.reconnWebSock, this.messageCount++,
                resultCallback, timeoutMS, jsonData, this.sockOnMessage);
            msgHandle.retryMax = this.defaultRetryMax;
            this.messagesWithCallback.push(msgHandle);
            this.reconnWebSock.send(jsonData);
        }
        ...
        ```

    - When the response message is received within the timeout period, the message handler calls the callback function, clears itself, and is deleted from the message list.

        ```TypeScript
        ...
        sockOnMessage(event: MessageEvent) {
            if (this.messagesWithCallback === null || this.messagesWithCallback.length == 0) {
                return;
            }
            const result: ApiResult = JSON.parse(event.data);
            const handlerIndex: number = this.messagesWithCallback.findIndex(MSG => MSG.msgID === result.msgID);
            if (handlerIndex < 0) {
                console.log("handlerIndex < 0 : sockOnMessage() : " + result.message);
                return;
            }
            this.messagesWithCallback[handlerIndex].dispose();
            const resultCallback = this.messagesWithCallback[handlerIndex].resultCallback;
            this.messagesWithCallback.splice(handlerIndex, 1);
            if (resultCallback) {
                resultCallback(result);
            } 
            this.checkDisposedMessageHandlers();
        }
        ...
        ```



