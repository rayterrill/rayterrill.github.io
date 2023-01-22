---
title:  "Adding Audio Output to an Alexa Skill"
tags: [aws]
---

It's pretty easy to add audio to an existing Alexa skill. The biggest trick is probably getting the audio converted into an Alexa-Friendly format. Here's how I did this:
1. Exported my audio from Adobe Premier (or whatever you're using to build your audio files)
2. Used ffmpeg to convert the audio to the right format:
```
ffmpeg -i my.mp3 -ac 2 -codec:a libmp3lame -b:a 48k -ar 16000 my-converted.mp3
```
1. Upload my new converted audio into an s3 bucket

### Adding Audio to an Alexa Skill

Assuming you already have an existing skill, you can use this simple SSML to refer to your file:
```
<audio src="https://s3-us-west-2.amazonaws.com/yourbucketname/my_converted.mp3" />
```
