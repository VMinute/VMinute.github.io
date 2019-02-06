---
layout: post
title: Azure IoT Edge on Torizon
date: 2012-02-06 12:00:00
tags: azure iot edge torizon
author: vminute
---
As you may have noticed, Toradex (my employer) has released a preview of a new Linux-based operating system named Torizon:  
[https://labs.toradex.com/projects/torizon](https://labs.toradex.com/projects/torizon)  
I also contributed to developing a Visual Studio extension that allows you to develop native Linux applications from a Windows PC:  
[https://labs.toradex.com/projects/torizon-microsoft-environment](https://labs.toradex.com/projects/torizon-microsoft-environment)  
(more on this in a future blog post, I hope)  
One of the interesting things about Torizon is that it allows you to deploy your applications as containers. I like the idea and spent quite some time playing with docker on one of my modules.  
Containers are already quite popular on the cloud and in web-development, but they are becoming very popular also in embedded.  
Many companies (ARM, RedHat etc.) have announced solutions that allow you to run containers on embedded devices. Torizon is one of those solutions and it's already available as a preview.  
Another interesting IoT application using containers is Azure IoT Edge.  
After having developed great connectivity, monitoring, and management solutions, Microsoft realized that many application can't rely on the cloud for all their "intelligence". Some processing has to happen on the device, at the "edge" of the system. Slow or unreliable connections, low latencies, and other factors make uploading data to the cloud and wait for a response an ineffective approach.  
And, surprise surprise, also Azure IoT Edge is based on containers.  
The runtime downloads and runs containers, updating its configuration dynamically from the cloud.  
Of course, I wanted to try it... but running the runtime requires adding some components to the OS image, build rust code etc.  
Looks like the typical scenario where containers could help, isn't it?  
So I decided to run also Azure IoT Edge runtime inside a container!
If you want to try this you can follow this tutorial:  
[https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux](https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux)  
But instead of installing the runtime directly on the running OS you can deploy it as a container, as described here:  
[https://github.com/VMinute/torizon-azure-iot-edge](https://github.com/VMinute/torizon-azure-iot-edge)
(this GitHub repo also includes the DockerFile I used to build my container).  
Just add the device connection string and ip address to config.yaml and you are ready to test Azure IoT from a real device!  
This is not something meant to be used in production, of course, but it's an easy way to "play" with Azure IoT Edge, reducing integration and configuration time.  
