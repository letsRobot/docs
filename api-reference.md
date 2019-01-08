# WORK IN PROGRESS #



# Robot API Documentation

## Introduction
Your robot's software has to do a few things:
 - Receive commands from users
 - Receive chat messages from users
 - Send out a video and audio stream
 - Tell the server it is ready to go LIVE

All of the communication between your robot and LetsRobot's servers is done over HTTP. To enable realtime, bi-directional communication between your robot and the servers we use [socket.io](https://en.wikipedia.org/wiki/Socket.IO) for the data messages. For the audio and video we use a [MPEG](https://nl.wikipedia.org/wiki/MPEG) stream.

In the examples we will be using code written in URL(Python) and use URL(ffmpeg) to stream audio and video.

## URLs
http://letsrobot.tv/get_websocket_relay_host/<camera-id>
http://letsrobot.tv/get_video_port/<camera-id>
http://letsrobot.tv/get_control_host_port/<robot-id>?version=2
http://<???robotid???>.robot-control.letsrobot.tv:<controlhostport>
http://letsrobot.tv/get_chat_host_port/<robot-id>
https://api.letsrobot.tv/api/v1/robots/<robot-id>
http://<relay-host>:<video-port>/<streamkey>/<width>/<height>/
http://letsrobot.tv:8022

## Receiving commands
First we need the control socket's host and port:
GET https://letsrobot.tv/get_control_host_port/12345
RESPONSE {"host":"<control-host>","port":<control-port>}
 
Which we can use to create the control socket's url:
http://<control-host>:<control-port>
 
After connecting to the socket,  our robot needs to identify itself:
controlSocket.Emit("robot_id", <robot-id>);

We then subscribe to the command event:
controlSocket.On("command_to_robot", OnCommandToRobot);

Response:
command: forward/up/down/left/whateveryouput
keystate: down
user: alice
....

## Receiving chat messages
text

## Streaming audio and video
GET camera id
GET relay host
GET video port
MPEG stream to URL using FFmpeg

## Going LIVE
Connect to socket
Emit robot id
Send video status signal
