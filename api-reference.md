
# ***WORK IN PROGRESS***

# Robot API Documentation

## Introduction

Your robot's software has to do a few things:
- Receive commands from users
- Receive chat messages from users
- Send out a video and audio stream
- Tell the server it is ready to go LIVE

All of the communication between your robot and [LetsRobot](https://letsrobot.tv/)'s servers is done over [HTTPS](https://en.wikipedia.org/wiki/HTTPS). We use [socket.io](https://en.wikipedia.org/wiki/Socket.IO) for the data communication, and an [MPEG](https://en.wikipedia.org/wiki/MPEG) stream for the audio and video.

In the examples we will use code written in [Python](https://en.wikipedia.org/wiki/Python_%28programming_language%29) and use [ffmpeg](https://en.wikipedia.org/wiki/FFmpeg) to stream audio and video. You can find an example of a fully implemented client [here](https://github.com/letsRobot/letsrobot).

You will need your robot-id, camera-id and streamkey, all of which you can find on your robot's settings page on your profile.

## Receiving control commands

### Constructing the control socket's URL

To construct the control socket's URL, we first need to fetch the control socket's host and port.

We can get these from a GET request to this URL:

```
https://letsrobot.tv/get_control_host_port/{robot-id}
```

Which will give us a JSON response:

```
{ 
	host: "control-host", 
	port: control-host-port 
}
```

From which we can construct the control socket's URL:

```
https://{control-host}:{control-host-port}
```

### Connecting to the control socket

Now that we have the URL we can connect to the socket:

```
from socketIO_client import SocketIO, LoggingNamespace

controlSocketIO = SocketIO(controlHost, controlHostPort, LoggingNamespace, transports='websocket')
```

After connecting to the socket, our robot needs to identify itself:

```
controlSocketIO.emit('robot_id', robotID)
```

And we need to subscribe to the command event:

```
controlSocket.on('command_to_robot', onHandleCommand)
```

### Handling received control messages

Every time a user clicks a control button on the website, we will receive a JSON message on the control socket:

```
{
	robot_id: "robot-id",
	command: "command",  
	user: "[empty|username]",  
	key_position: "[empty|down|intermediate|up]",  
	anonymous: [true|false]
}
```

Which we handle in our onHandleCommand function:

```
def onHandleCommand(*args):
	print('got command', args)
	command = args['command']
```

## Receiving chat messages

### Constructing the chat socket's URL

To construct the chat socket's URL, we first need to fetch the chat socket's host and port.

We can get these from a GET request to this URL:

```
https://letsrobot.tv/get_chat_host_port/{robot-id}
```

Which will give us a JSON response:

```
{
 host: "chat-host",
 port: chat-host-port
}
```

From which we can construct the chat socket's URL:

```
https://{chat-host}:{chat-host-port}
```

### Connecting to the chat socket

Now that we have the URL we can connect to the socket:

```
from socketIO_client import SocketIO, LoggingNamespace

chatSocket = SocketIO(chatHost, chatHostPort, LoggingNamespace, transports='websocket')
```

After connecting to the socket,  our robot needs to identify itself:

```
chatSocket.emit('identify_robot_id', robotID)
```

And we need to subscribe to the chat event:

```
chatSocket.on('chat_message_with_name', onHandleChatMessage)
```
 

### Handling received chat messages

Every time a user sends a chat message on the website, we will receive a JSON message on the chat socket:

```
{
	name: "username",
	message: "message",
    robot_id: "robot-id",
    room: "roomname",
    non_global: [true|false],
    username_color: "#000000",
    anonymous: [true|false],
    _id: "6c46183223732206e58e27f9"
}
```

Which we handle in our onHandleChatMessage function:

```
def onHandleChatMessage(*args):
	print('got chat message', args)
	message = args['message']
```

## Streaming audio and video

The audio and video are streamed separately, so we will have to setup two streams.

### Constructing the video stream's URL

To construct the video stream's URL, we first need to fetch the video stream's host and port.

We can get these from GET requests to these URLs:

```
https://letsrobot.tv/get_websocket_relay_host/{camera-id}
```

```
https://letsrobot.tv/get_video_port/{camera-id}
```

Which will give us these JSON responses:

```
{ host: "relay-host" }
```

```
{ port: video-host-port }
```

From which we can construct the video stream's URL:

```
https://{relay-host}:{video-host-port}/{streamkey}/{xres}/{yres}/
```

### Constructing the audio stream's URL

To construct the audio stream's URL, we first need to fetch the audio stream's host and port.

We can get these from GET requests to these URLs:

```
https://letsrobot.tv/get_websocket_relay_host/{camera-id}
```

```
https://letsrobot.tv/get_audio_port/{camera-id}
```

Which will give us these JSON responses:

```
{ host: "relay-host" }
```

```
{ port: audio-host-port }
```

From which we can construct the audio stream's URL:

```
https://{relay-host}:{audio-host-port}/{streamkey}/{xres}/{yres}/
```

### Streaming video

To stream video, we need to send MPEG1 encoded video in an MPEG-TS container to the video stream URL.

To do this, we can use ffmpeg:

```
ffmpeg -f v4l2 -video_size {xres}x{yres} -i /dev/video{video-device-number} -f mpegts -framerate 25 -codec:v mpeg1video -b:v {kbps}k -bf 0 -muxdelay 0.001 https://{relay-host}:{video-port}/{streamkey}/{xres}/{yres}/
```

### Streaming audio

To stream audio, we need to send MP2 encoded audio in an MPEG-TS container to the audio stream URL.

To do this, we can use ffmpeg:

```
ffmpeg -f alsa -ar 44100 -ac {audio-device-number} -i hw:{audio-device-number} -f mpegts -codec:a mp2 -b:a 32k -muxdelay 0.001 https://{relay-host}:{audio-port}/{streamkey}/{xres}/{yres}/
```

## Going LIVE

To tell the server that your bot is ready to be marked as LIVE, we need to send a video status signal.

To do this, we first connect to the video status socket:

```
from socketIO_client import SocketIO, LoggingNamespace

videoStatusSocket = SocketIO('letsrobot.tv', 8022, LoggingNamespace, transports='websocket')
```

After connecting to the socket,  our robot needs to identify itself:

```
videoStatusSocket.emit('identify_robot_id', robotID)
```

After identifying, we have to send the video status message every 30 seconds:

```Python
videoStatusSocket.emit(
	'send_video_status',
	{
		'send_video_process_exists': True,
		'camera_id': cameraID
	}
);
```

Your robot is now LIVE and ready to receive user input.

## Various

### Getting the robot's owner

We can get the robot's owner from a GET request to this URL:

```
https://letsrobot.tv/get_robot_owner/{robot-id}
```

Which will give us this JSON response:

```
{ owner: "owner-name" }
```

You can use this to determine wether commands are being sent by the owner.

### Getting the robot's details and settings

We can get the robot's details and settings from a GET request to this URL:

```
https://api.letsrobot.tv/api/v1/robots/{robot-id}
```

Which will give us this JSON response with the robot's details and settings.

You can use this to retrieve the settings you have set on your robot's settings page on your profile, and adjust the robot's behaviour accordingly.

### User socket

Info about user socket goes here.
```
https://letsrobot.tv:8000
```

### Woot room

Info about woot room goes here.

### Get WiFi settings

Getting wifi settings goes here.

### Text-to-Speech

Python Text-to-Speech example goes here.

### Exclusive control

Info about exclusive control goes here.

### Update robot settings

Info about updating robot settings through API goes here.

### Start Reverse SSH

WTF

```
appServerSocketIO.on('reverse_ssh_8872381747239', startReverseSshProcess)
appServerSocketIO.on('end_reverse_ssh_8872381747239', endReverseSshProcess)
```

### Robot is charging

Info about robot is charging here

### Reconnecting on connection loss

Don't forget to add code to have your client reconnect the sockets and stream after a connection loss.

## List of all socket events

List of all socket events goes here.

```
controlSocketIO.on('command_to_robot', onHandleCommand)
controlSocketIO.on('disconnect', onHandleControlDisconnect)
controlSocketIO.on('connect', onHandleControlConnect)
controlSocketIO.on('reconnect', onHandleControlReconnect)

appServerSocketIO.on('exclusive_control', onHandleExclusiveControl)
appServerSocketIO.on('connect', onHandleAppServerConnect)
appServerSocketIO.on('reconnect', onHandleAppServerReconnect)
appServerSocketIO.on('disconnect', onHandleAppServerDisconnect)
appServerSocketIO.on('command_to_robot', onCommandToRobot)
appServerSocketIO.on('connection', onConnection)
appServerSocketIO.on('robot_settings_changed', onRobotSettingsChanged)

userSocket.on('message_removed', onHandleChatMessageRemoved)

if commandArgs.woot_room != '':
userSocket.on('chat_message_with_name', onHandleUserSocketChatMessage)

chatSocket.on('chat_message_with_name', onHandleChatMessage)
chatSocket.on('connect', onHandleChatConnect)
chatSocket.on('reconnect', onHandleChatReconnect)
chatSocket.on('disconnect', onHandleChatDisconnect)

appServerSocketIO.on('reverse_ssh_8872381747239', startReverseSshProcess)
appServerSocketIO.on('end_reverse_ssh_8872381747239', endReverseSshProcess)

def sendChargeState():
	charging = isCharging()
	chargeState = {'robot_id': robotID, 'charging': charging}
	appServerSocketIO.emit('charge_state', chargeState)
	print "charge state:", chargeState
```
