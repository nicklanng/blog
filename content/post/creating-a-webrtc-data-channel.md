+++
author = "Nick Lanng"
categories = ["code", "node", "javascript"]
date = 2016-04-19T15:06:08Z
description = ""
draft = false
slug = "creating-a-webrtc-data-channel"
tags = ["code", "node", "javascript"]
title = "Creating a WebRTC Data Channel"

+++

I was recently learning about browser-based game development and was wondering about multiplayer games. WebSockets are super easy to work with but are they any good for games? The answer is yes and no. If you make a turn-based thing that doesn't require up to the millisecond updates then WebSockets are the way to go, but for any fast-paced games they're just not up to scratch.

WebSockets are built on TCP, a reliable messaging protocol that ensures all data arrives and that it arrives in order. The repercussions of this is a short delay whenever a message goes missing, all the messages that come after have to wait for the missing one to be resent and received before they can be processed.

WebRTC is a newer technology that allows peer to peer transmission of three channels: audio, video and data. It was the data channel that interested me. WebRTC transmits over UDP, it has to or the audio and video channels would have super choppy playback while waiting for any missed packets to be resent. If I could create a WebRTC data channel from the browser to a server then I could have fast UDP game messages and hopefully get a better multiplayer experience.

As always with Node, this was so much simpler than I was expecting! Below is an explanation of the important bits of code. You can click the button in the top right to go to the Github repository for this project but be warned that the server doesn't support Windows; this is due to the **wrtc** library that mimics some WebRTC features only found in browsers.

# How does it work?

In order to create a WebRTC connection, the two legs of the connection need to be able to signal each other. Without an existing WebRTC connection you need some sort of reliable messaging system, WebSockets is perfect for this! We use WebSockets to allow the two legs of the connection to signal and negotiate a WebRTC connection.

The initiating leg sends an offer to its requested party. In this case, the initiator is the web page and it offers a connection to the server. The server responds to the offer with an accept. All of this communication is done over WebSockets.

Once both parties have agreed to connect and have been told how to connect to each other, a WebRTC connection is established.

The actual process of signalling is a little more complex than that, with STUN and TURN servers and ICE. Lots of scary acronyms, but the basics are that these servers let each client know how the other can connect to it through NATs and the like, and then the clients tell each other that information .

### NPM Packages
There are three important packages used in the solution.

* [socket.io](https://www.npmjs.com/package/socket.io) WebSockets for Node.js and the browser
* [simple-peer](https://www.npmjs.com/package/simple-peer) Simple WebRTC video/voice and data channels
* [wrtc](https://www.npmjs.com/package/wrtc) WebRTC stack for node.js

# The code
### Client

{{< highlight js>}}
// main.js
import SimplePeer from 'simple-peer'

// create the WebSocket connection
const socket = io()

// create a WebRTC connection, as the client this will be the initiating leg
const peer = new SimplePeer({ initiator: true })

// when we get a signal, send it over the WebSocket to the server
peer.on('signal', (data) => socket.emit('signal', data))

// WebRTC connection is successful!
peer.on('connect', () => console.log('connected'))

// when we get a signal over the WebSocket from the server, signal the WebRTC connection
socket.on('signal', (data) => peer.signal(data))
{{< /highlight >}}

### Server
{{< highlight js>}}
// index.js
import express from 'express'
import { Server } from 'http'
import SocketIo from 'socket.io'
import connection from './connection'

// create the web server with a WebSocket connection
var app = express()
var server = Server(app)
var io = SocketIo(server)

app.use(express.static('static'))

// once we connect over WebSocket, try to start a WebRTC connection
io.on('connection', connection)

server.listen(3000, () => console.log('listening on *:3000'))
{{< /highlight >}}

{{< highlight js>}}
// connection.js
import SimplePeer from 'simple-peer'

// bring in our WebRTC polyfill
import wrtc from 'wrtc'

// this code is 90% the same as the code in the client!
const connection = (socket) => {
  // be sure to pass in the WebRTC polyfills
  const peer = new SimplePeer({ wrtc })

  // when we get a signal, send it over the WebSocket to the server
  peer.on('signal', (data) => socket.emit('signal', data))

  // WebRTC connection is successful!
  peer.on('connect', () => console.log('connected'))

  // when we get a signal over the WebSocket from the client, signal the WebRTC connection
  socket.on('signal', (data) => peer.signal(data))
}

export default connection
{{< /highlight >}}

# Sending data
I'm not sending any data across in the example project, so let me show you here how I like to do it.

A simple-peer connection has a function called **send**. This function takes just one argument which is the data you want to send. The supported data types are **String**, **Buffer**, **TypedArrayView** ([more specifically, those types that extend it](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays)), **ArrayBuffer** or **Blob** (where supported).

{{< highlight js>}}
peer.send('I am sending a string!')
peer.send(32) // WATCH OUT - Gets converted to a string
peer.send(new Uint8Array([32]))
peer.send(new Buffer([32]))
{{< /highlight >}}

I favor using Buffer, this is a common type in Node.js and the creator of Simple-Peer has also created a [version of Buffer that works in the browser](https://github.com/feross/buffer). This way, I can use the same netcode across my server and my client.

To receive data, simply implement on a data event handler!
{{< highlight js>}}
peer.on('data', (data) => console.log(data))
{{< /highlight >}}
