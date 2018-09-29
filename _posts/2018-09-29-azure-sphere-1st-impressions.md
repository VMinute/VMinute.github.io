---
layout: post
title: Azure Sphere first impressions
date: 2018-09-28 12:00:00
tags: azure sphere iot
author: vminute
---
I got an Azure Sphere kit a few weeks ago and I had a chance to experiment using it during my free time.  
The kits are now available and the software is in public beta and, even if it's still lacking many features, it's a great way to start experimenting with this new technology.  
Documentation has been released a few weeks before the official announcement and it provides a good idea of how the system works (I think it's not correct to think about Azure Sphere just as a device since it involves also cloud-based services) and on how to set-up your board and development environment. You can find it on <a href="https://docs.microsoft.com/en-us/azure-sphere/">https://docs.microsoft.com/en-us/azure-sphere/</a>.  
As you can see from the architecture description, Azure Sphere provides a mix of hardware and software components designed to be the low-level building blocks of a secure system made of small devices.  
Currently, most of those devices run on microcontrollers and security (in terms of secure connections and updates fixing potentially exploitable bugs) is a big issue that already led to some embarrassing incidents. We have seen "botnets" of hacked devices used to perform denial of service attacks or apparently harmful smart connected object being used to access secure networks (I don't know if it's fully proven, but <a href="https://www.businessinsider.com/hackers-stole-a-casinos-database-through-a-thermometer-in-the-lobby-fish-tank-2018-4?IR=T">this story sounds crazy and not too far from being plausible</a>). As we are going to see more "smart" (used a synonymous of "connected") devices appearing in our houses, offices, working environments, cities, securing those devices will become a critical issue that can potentially impact the growth of this large market and limit the usefulness of those technologies for their end users.  
Azure Sphere is interesting because it tries to address the issue integrating all the components of the IoT stack, from the hardware up to the cloud-based services (even if Azure Sphere is not tied to Microsoft own Azure cloud for the application backend services). This involves hardware IPs (Pluton), an operating system that provides a secure "sandbox" for applications and can be updated over the air, a secure wi-fi stack and a set of cloud-based services that can support a large number of devices on the field.
## Azure Sphere HW and SW on the device
Currently, there is only one System On Chip designed to run Azure Sphere application: MediaTek's MT3620. From Microsoft announcements, is clear that this is not the only platform that will support this architecture and the hardware component that implement the low-level security features of Azure Sphere (Pluton) will be available for any SOC vendor willing to integrate it in their chips.  
The MediaTek SOC is using ARM cores and it's something between a microcontroller (like those you can find in simple devices like your washing machine) and an application processor (like the one you can find in your internet router). It has a Cortex-A7 core that runs a custom version of Linux (Linux on a Microsoft product! How crazy that would have sounded just five years ago?) and additional Cortex-M4 cores that can run low-latency dedicated applications. Also the amount of memory and storage it provides (4MB of RAM and 16MB of flash, both integrated in the chip, as usually happens with microcontrollers) and the main CPU clock speed (500MHz) are somewhat in between an high-end microcontroller and a low-end application processor (where usually storage and RAM are external).
On Azure Sphere, the Linux kernel and user-mode components are used to provide a sort of "sandbox" for the application that is a packaged in a format that allows over-the-air updates and will simplify distribution on a large number of devices. Of course, there is no "marketplace" or features to allow end-users to download applications on their devices, this makes sense considering that the system targets small and usually single-purpose specialized devices.
## Installing and configuring Azure Sphere
Azure Sphere applications can be developed using Visual Studio 2017. This just require the installation of an additional extension named <a href="https://aka.ms/AzureSphereSDKDownload">Azure Sphere SDK for Visual Studio</a>. This will allow you to create new Azure Sphere applications and libraries.  
But to be able to download your applications to your brand new Azure Sphere device you'll have to connect it to an Azure account first. To be completely honest you'll also have to physically connect the board to your device before, but this is just a matter of plugging in a USB cable and wait until some drivers are installed.  
As I said Azure Sphere involves more than the hardware and to be able to use your device and allow it to download OS updates (that are part of what you get when you buy the MT3620 chip or a development board).  The process is, at the moment, far from being intuitive and involves a number of command line operations. I had a few issues doing that from my main azure account, but creating a new user (still connected to the same billing account), solved the issue. The only suggestion I have is to carefully follow the <a href="https://docs.microsoft.com/en-us/azure-sphere/install/azure-directory-account">instructions provided on the documentation website</a>. Be warned that this is a one-time operation, once you claim an Azure Sphere device for your own account it will be connected to it forever. This will let you manage your devices and let them download updates that can be critical to keep them secure over time.  
Once you have finished setting up the device and development environment you can start developing your applications.
## Application development on Azure Sphere
At the moment (and probably also in the long term, considering the limited amount of resources available on those devices) the only development language supported on Azure Sphere is C.  
You read that correctly, it's plain old C. This may sound a bit scary for people used to high-level languages and environments, even on devices, but seems to be a pretty rational choice given the constraints and the target market. C is still the most popular language for development on microcontrollers and even the very popular Arduino IDE actually uses C with just a bit of C++ (limited to very basic classes, some syntax and far from being the complete language used on desktop and server development).  
Azure Sphere provides its own API for hardware access and connectivity and relies on standard Linux APIs like pthreads for other features. On the other side, you shouldn't expect a full-blown Linux development environment and the C library functions that are currently available don't include, for example, something very basic like folders navigation. Even if your application will run as a Linux process it will be "shielded" from accessing the OS directly. Each application has a manifest (a concept that is used also in application development on Android, for example, where apps should advertise the features they use and, for some of them, require an explicit permission from the user to access them) that defines what hardware and connectivity resources it will access. The system will prevent buggy or compromised/malicious apps from accessing resources that they shouldn't use for their normal operations.  
<a href="https://docs.microsoft.com/en-us/azure-sphere/app-development/app-manifest">More informations on the manifest is available in the documentation</a>.  
The tools, even in this early beta, are very well integrated into Visual Studio. Most of the editing, project management, debugging features work seamlessly and you will be able to download and deploy an application without much effort.  
You can try to create a simple "hello world" application to see how it's structured. For embedded devices instead of printing "hello world" on some kind of screen or console in your first sample app, it's normal to blink an LED and Azure Sphere is no exception. You can create a blink application for the MT3620 board (applications are hardware-specific and this makes sense considering the tight integration between hardware and software on devices) and you will have some files automagically generated for you.  
The manifest will be named app_manifest.json and will look like this:
```json
{
  "SchemaVersion": 1,
  "Name" : "Mt3620Blink1",
  "ComponentId" : "91e246b0-b9c9-409d-9313-9b6a660a32f3",
  "EntryPoint": "/bin/app",
  "CmdArgs": [],
  "TargetApplicationRuntimeVersion": 1,
  "Capabilities": {
    "AllowedConnections": [],
    "Gpio": [ 8, 9, 10, 12 ],
    "Uart": [],
    "WifiConfig": false
  }
}
```
It will provide a name and unique id for your app and then it will specify some capabilities. Those can be used to define which resources your application will be able to access.  
Looking at the contents of main.c and starting from the bottom we could see the main entry point that looks quite typical for a standard C application:
```C
/// <summary>
///     Main entry point for this application.
/// </summary>
int main(int argc, char *argv[])
{
    Log_Debug("Blink application starting\n");
    if (InitPeripheralsAndHandlers() != 0) {
        terminationRequired = true;
    }

    // Use epoll to wait for events and trigger handlers, 
    // until an error or SIGTERM happens
    while (!terminationRequired) {
        if (WaitForEventAndCallHandler(epollFd) != 0) {
            terminationRequired = true;
        }
    }

    ClosePeripheralsAndHandlers();
    Log_Debug("Application exiting\n");
    return 0;
}
```
The platform provides logging capabilities and during debug logging messages are automatically routed to Visual Studio debug window.  
As you can see, apart from the main prototype, from the architecture point of view this code looks more like a traditional microcontroller firmware than like a Linux program. We have some hardware initialization and then the code loops waiting for the termination signal or to service some timers. This will probably let the CPU free to sleep and save power for most of the time the application run, saving some power.  
As you can suspect the most interesting parts of the code are inside the **InitPeripheralsAndHandlers** function.
First, a termination handler function is assigned to the SIGTERM signal. Signals are a mechanism that Unix-like OSs use to request actions from applications. In this case, the OS will require termination and give our app a chance to leave the hardware in a reliable state (think about an oven, turning off heating elements may sound like a safe thing to do there).
```C
    struct sigaction action;
    memset(&action, 0, sizeof(struct sigaction));
    action.sa_handler = TerminationHandler;
    sigaction(SIGTERM, &action, NULL);
```
A timer is then created and, as typical for unix, a file descriptor (fd) is returned. Remember, "everything is a file", even if it doesn't look like one!  
```C
    epollFd = CreateEpollFd();
    if (epollFd < 0) {
        return -1;
    }
```
Then we start initializing the hardware, just GPIO pins in this case.
```C
    // Open button GPIO as input, and set up a timer to poll it
    Log_Debug("Opening MT3620_RDB_BUTTON_A as input\n");
    gpioButtonFd = GPIO_OpenAsInput(MT3620_RDB_BUTTON_A);
    if (gpioButtonFd < 0) {
        Log_Debug("ERROR: Could not open button GPIO: %s (%d).\n", 
            strerror(errno), errno);
        return -1;
    }
    struct timespec buttonPressCheckPeriod = {0, 1000000};
    gpioButtonTimerFd = CreateTimerFdAndAddToEpoll(epollFd, 
        &buttonPressCheckPeriod,
        &ButtonTimerEventHandler, EPOLLIN);
    if (gpioButtonTimerFd < 0) {
        return -1;
    }

    // Open LED GPIO, set as output with value GPIO_Value_High (off), 
    // and set up a timer to poll it
    Log_Debug("Opening MT3620_RDB_LED1_RED\n");
    gpioLedFd = GPIO_OpenAsOutput(MT3620_RDB_LED1_RED,
        GPIO_OutputMode_PushPull, 
        GPIO_Value_High);
    if (gpioLedFd < 0) {
        Log_Debug("ERROR: Could not open LED GPIO: %s (%d).\n", 
            strerror(errno), errno);
        return -1;
    }
     gpioLedTimerFd = CreateTimerFdAndAddToEpoll(epollFd, 
        &blinkIntervals[blinkIntervalIndex],
        &LedTimerEventHandler, EPOLLIN);
    if (gpioLedTimerFd < 0) {
        return -1;
    }
```
In this case, instead of using the /sys filesystem to access and configure GPIO pins as you would normally do on a Linux device, some Azure Sphere-specific APIs are called to configure the pins. Notice that the code uses MT3620-specific constants. Probably you'll need some macro magic or configuration parameters to write code that will be compatible with different boards, but this is true for most microcontroller-based solutions.
Instead of polling the input button and changing the state of the output LED inside a loop in main (as Arduino sketches typically do), specific handler functions are associates to periodic timers.  
It reminds me of the "event-driven" application model of traditional Win32 applications (people who started developing on Windows before MFC, .NET or any other framework was invented will probably remember that...and maybe still feel the pain), but makes sense for simple and resource-constrained apps.  
The LED timer event handler will just toggle the LED state using the GPIO APIs.  
LED state is kept in a global variable and, even if this will make people used to advanced OOP designs scream and pull their hair, it's quite common practice in firmware and was quite common also in old Windows apps (before they start getting too complex for that kind of programming model).
```C
/// <summary>
///     Handle LED timer event: blink LED.
/// </summary>
static void LedTimerEventHandler()
{
    if (ConsumeTimerFdEvent(gpioLedTimerFd) != 0) {
        terminationRequired = true;
        return;
    }

    // The blink interval has elapsed, so toggle the LED state
    // The LED is active-low so GPIO_Value_Low is on and GPIO_Value_High is off
    ledState = (ledState == GPIO_Value_Low ? GPIO_Value_High : GPIO_Value_Low);
    int result = GPIO_SetValue(gpioLedFd, ledState);
    if (result != 0) {
        Log_Debug("ERROR: Could not set LED output value: %s (%d).\n", strerror(errno), errno);
        terminationRequired = true;
    }
}
```
The button event handler is slightly more complex, but noting too hard to understand:  
```C
/// <summary>
///     Handle button timer event: if the button is pressed, change the LED blink rate.
/// </summary>
static void ButtonTimerEventHandler()
{
    if (ConsumeTimerFdEvent(gpioButtonTimerFd) != 0) {
        terminationRequired = true;
        return;
    }

    // Check for a button press
    GPIO_Value_Type newButtonState;
    int result = GPIO_GetValue(gpioButtonFd, &newButtonState);
    if (result != 0) {
        Log_Debug("ERROR: Could not read button GPIO: %s (%d).\n", 
            strerror(errno), errno);
        terminationRequired = true;
        return;
    }

    // If the button has just been pressed, change the LED blink interval
    // The button has GPIO_Value_Low when pressed and GPIO_Value_High 
    // when released
    if (newButtonState != buttonState) {
        if (newButtonState == GPIO_Value_Low) {
            blinkIntervalIndex = (blinkIntervalIndex + 1) % numBlinkIntervals;
            if (SetTimerFdInterval(gpioLedTimerFd, 
                &blinkIntervals[blinkIntervalIndex]) != 0) {
                terminationRequired = true;
            }
        }
        buttonState = newButtonState;
    }
}
```
Pressing the button will change the LED blinking frequency working on the LED timer as, again, global variable.  
As you can see both event handler "consume" the timer event, allowing a new one to be generated.  
Code is not too complex, and some additional things are generated inside the project (helper functions to simplify timer management and a platform-specific header file with mnemonic defines for all the pins and device you may use in your code) keeping it simple enough to be easily understood also by developers who never worked on Linux or embedded devices.  
What is really nice is what happens when you press F5 to start your new application.
Your application will be built and copied to the device, you will be able to set breakpoints, inspect variables, step through your own code etc.  
Debugging uses remote GDB, the most commonly used Linux debugger. As it happens with the Linux development extension for Visual Studio, you can run an debug a remote native Linux application for the most "microsoftish" tool ever made! This is so cool.
Some Azure Sphere specific tools are used to "package" our app and deploy it to the target, but looking at the content of the SDK installed under "C:\Program Files (x86)\Microsoft Azure Sphere SDK" we can discover something interesting.  
In the "tools" folder you can find azsphere.exe that is the tool you already used to connect the device to your Azure account and that can be used to change its configuration or to deploy applications via USB or over the air.  
We have a folder named "sysroot", and this sounds quite familiar. Sysroots are normally used when cross-compiling applications. You may want to cross-compile when your development machine uses a different architecture or OS version compared to your target hardware and compiling on the target is not practical due to its limitations in terms of processing power, RAM and/or storage. A sysroot provides include files and libraries that map the features available on the target device.   
In the "sysroot/tools" folder we can find our cross-compiler. It's, unsurprisingly, gcc (unsurprisingly since it's quite common, a bit surprising because it's provided directly by Microsoft, but you should be used to this at this point). It's name:  "arm-poky-linux-musleabi-gcc.exe" or, better, the prefix it shares with all the other build tools, tells us many interesting things.  
Let's start from the beginning. It's an arm compiler. This is not surprising considering that all CPU cores inside our target soc are ARMs. It's also not unsurprising that we see a reference to Linux, since this is our target os.  
The name poky will ring a bell for people that work on embedded Linux devices. It's the name of one of the reference distributions of the Yocto project. Yocto is probably the most popular development environment for embedded Linux (used also at <a href="www.toradex.com">Toradex</a>, by the way) and this may be a hint that also Microsoft own custom Linux is Yocto-based (not being able to check the actual filesystem this is just an hypothesis). Musl is a relatively recent but already widely used implementation of the standard C library that aims to replace glibc (the library most commonly used on Linux desktop and server variants) on embedded devices, providing a complete implementation that uses less memory and generates more compact code.
It seems that after years spent developing his own embedded OSs (Windows CE first and Windows 10 IoT core) Microsoft has chosen some of the most popular solutions in embedded for this new product. This does not mean that development of Windows 10 IoT core has been cancelled, but for sure it's a sign that in Redmond people no longer consider only OSs with Windows in their name.  
On the other side, the quality of the development tools is up to the usual Microsoft standards (I know that not everyone likes Visual Studio, I do and everyone who does will appreciate how well Azure Sphere is integrated inside the IDE).  
## Sample code
I wanted to play with Azure Sphere and decided to build a small library to implement software PWM. This is not the best application for a non-realtime OS like Linux and on a feature-complete version of Azure Sphere this will probably be handled directly in hardware or, if you need additional channels, on the M4 cores. At the moment the toolkit does not support hardware PWM or the extra cores, so I implemented it using threads and GPIOs.  
After that, I decided to connect a small servo motor to have a more "visual" feedback of my library working and so I made a simple servo motor control library and a demo that spins a motor connected to the MT3620 board.  
<a href="https://github.com/VMinute/AzureSphereSamples">You can find my code on GitHub in a project named AzureSphereSamples</a>.  
Here you can see the servo motor spinning:  
<iframe width="560" height="315" src="https://www.youtube.com/embed/1NsBdBeLHT0" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
## My two cents
Azure Sphere is a very interesting product and it addresses the need to develop simple devices that need to be securely connected to the Internet of Things.  
Traditional microcontrollers may be too limited to support all the security layers, device management and OTA update capabilities and systems running a full-blown OS (like those that could run Azure IoT Edge) may be too complex, expensive or power hungry.  
Azure Sphere tries to provide a solution made by hardware, operating system and cloud-based services all integrated in a secure way, this should be the basic architecture of an IoT solution and having security granted on the whole stack is definitely a big advantage.  
Of course, only time could tell how effective this solution will be, but Microsoft made a big step into this field providing easy to use development tools, a good hardware in terms of capabilities and price and committing to provide long-term service. A bit surprising this came with Linux, but this can be seen as a pragmatic approach, embracing what the market has declared the de-facto standard for devices running an operating system.  
The tools and core features are still a work in progress. Tools already seem to be quite reliable and provide most of the expected features but the libraries provided in this first beta cover just a small subset of the SOC capabilities and can't be used to implement a non-trivial device unless everything you need in term of hardware interfaces is GPIOs and serial ports.

