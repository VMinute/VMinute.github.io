---
layout: post
title: Building an IoT Device (part II)
date: 2018-07-29 12:00:00
tags: arduino esp32
author: vminute
---

Any connected device should provide or consume some data, otherwise being connected to the internet would be quite pointless, isn’t it?

My clock is connected to wi-fi (that’s why I choose ESP-32 as my platform), but a connection to a local network is not enough.

The clock has to collect some information from the net:
- current time (to not have to adjust it manually)
- information about events (to show them in the right way)
- images (to show nice icons for the different events)
- dates of holidays in a specific place (to use holidays wake-up times also for holidays that are in the middle of the week)

To collect those bits of information we will have to use different protocols and data formats. 

Existing Arduino libraries will help for some things, but I had to write some code to handle more project-specific stuff.

Let’s describe each item and how it is implemented in the software.

The code is available on GitHub here, so feel free to use it as a reference while reading this article, I’ll dig a bit deeper in the code in the next part, in this one I’ll try to provide an overview of how my small device interacts with the big network.

## Current Time

The ESP-32 module I used has no battery-powered RTC, this means that every time it powers up its date is set always to the same day. 
If this reminds you of “Groundhog Day” that is perfectly fine. 
If it doesn’t, go watch one of the funniest movies ever made!

If you are connected to the internet you can sync your internal clock with very precise atomic clocks from all around the world using the NTP protocol. This looks like a perfect solution to our problem! 
The easiest way to use it is to reference “pool.ntp.org” as your time server. This, via some DNS magic, will use different servers all around the world, granting redundancy and accuracy of the time information returned. 
There are some tricks to increase the accuracy of the time, considering network latencies, using servers geographically close to your location etc. but the easiest setup provides a quite accurate time for our purpose.

But, as usually happens in the real world, there are some glitches. 
Current time is not the same all over the world. We have time-zones, so 11 o’clock in Rome is not 11 o’clock in Sidney. 
So if you want to have a beer at any time of the day, you can say that it's the right time for a beer, somewhere in the world!
Probably you are already familiar with this issue, but if you think that it’s just a matter of adding a time offset you should reconsider your opinion. It’s a matter of adding an offset, but this may change during the year because many countries have daylight saving times and rules to apply them change from country to country. 

Luckily the C library provided for ESP-32 has a quite good support for time-zones and, once you configured the right time zone, having your internal RTC set to the right local time is a matter of a single Arduino API call: *configTzTime*.

You will have to provide a time-server (we have seen that pool.ntp.org is the best one we can use in this application) and a string describing your time zone.

Since rules for entering/leaving DST and setting time offset are quite complicated, someone designed a very complicated way to express those as strings.
Luckily Mr. Pavel Gurenko thought about all the lazy people like me who don’t want to read long specs to generate their time-zone strings and provided a <a href="http://www.pavelgurenko.com/2017/05/getting-posix-tz-strings-from-olson.html">very nice tool to generate them from your geographic location</a>. 
In this way by just selecting my location ("Europe/Rome”) I can get the full time-zone string: “Europe/Rome (CET-1CEST,M3.5.0,M10.5.0/3)”. 
The part between parenthesis is used to determine the right offset, depending on current date.

The time-zone string is stored in the device flash as configuration parameter (you will learn more about this in next part).

Now our clock has always the right local time. No more groundhog days!

## Calendars

The device should be able to collect information about events happening at a specific date. 
There are many online tools that you can use to manage your calendars like Google Calendar or Microsoft Outlook. 
You will have to define a specific calendar for the events displayed by the clock.
Luckily there is a standard format for calendar information named iCalendar (ICS). ICS is a text format and has a quite rich syntax that allows you to define recurring events etc. (like those nice weekly meetings at work). Parsing all those things is a bit above the target of our device so it will parse only fixed events.
The calendar is defined using a series of objects with their own properties. We have a calendar and inside it we have events.
The BEGIN:CALENDAR and END:CALENDAR tags are used to define beginning and end of a calendar (our code takes for granted that our ICS data contain a single calendar since this is true in most of the cases).
Calendars could have multiple properties, some standard ones and some added by specific apps that we could safely ignore.
The only property we use if X-WR-CALDESC that we use to collect default settings for wake-up and bedtime for regular days and holidays.
This can be a sample calendar header:
```
BEGIN:VCALENDAR
PRODID:-//Google Inc//Google Calendar 70.9054//EN
VERSION:2.0
CALSCALE:GREGORIAN
METHOD:PUBLISH
X-WR-CALNAME:My Calendar
X-WR-TIMEZONE:Europe/Rome
X-WR-CALDESC:N:08:00\nH:08:30\nB:21:30
...
END:VCALENDAR
```
As you can see we have information about the software/website that generated it, the time zone and in the X-WR-CALDESC settings about "normal" wake-up time (N), Holidays wake-up time (H) and bedtime (B). Those entries are on separate lines in the editor and separated by line-feed characters in the "C" language notification ('\n').
Inside a calendar you have multiple events like this one:
```
BEGIN:VEVENT
DTSTART:20171218T080000Z
DTEND:20171218T090000Z
DTSTAMP:20171227T150253Z
UID:6ifuo0574si2ghssf3mfak5a85@google.com
CREATED:20171223T145908Z
DESCRIPTION:Rugby
LAST-MODIFIED:20171223T145908Z
LOCATION:
SEQUENCE:0
STATUS:CONFIRMED
SUMMARY:Match in Milano
TRANSP:OPAQUE
END:VEVENT
```
This is a simple, single-instance event. 
You can also have recurring events (like every Monday, or every 10th of the month, or every 2nd Monday of the month), but those are not considered by our device.
Our code will use DTSTART and DTEND to calculate start and end time of the event. Start time will also be a wake-up time for that day.
As you can see dates are expressed in the ISO 8601 format (the only one that should be used to exchange date/time information between computers!), our code will have to translate them to local time but the ESP-32 C libraries provide easy to use functions for that.
We also use SUMMARY field to provide a short description of our event. The DESCRIPTION field is used to associate an icon for the event, we will see how images are managed in the next chapter of this article.

iCal format is used also for the holidays' calendar. 
You can find many websites that allow you to download holidays for your specific country/region/city in this format. In this case, the device will use only DTSTART, DTEND, and SUMMARY fields to collect information about the event.
Any event in the user calendar will override standard holidays.
The negative aspect of iCal, in my opinion, is that it does not use a standard format like XML or JSON, preventing usage of existing libraries to parse it (even if some of those libraries may not fit inside the limited amount of code and memory available on our device), on the other side writing a parser is not too complex, if you don't take into account repeated events and you can easily ignore properties that you don't know how to parse.

## Images

Since our device has a nice graphical display it would be nice to show some icons to make it easy to identify the kind of event that will be the "highlight" of a specific day. Some icons are embedded inside the code like those for a "regular" school/kindergarten day or for holidays. 
I wanted to be able to add new icons without having to update the code on the device, so I needed a way to add custom icons to events and download an image that I could use on the small monochrome screen.
I decided that icons were going to take a 48x48 pixel area on the 128x64 screen and they would be, of course, black and white, as our screen.
The event DESCRIPTION standard iCal field looked like the perfect place for the icon since SUMMARY was more than enough to provide an on-screen description of the event (limited to 2 lines of 16 characters each).
My first idea was to provide a URL pointing to the icon directly in the description, but that would have made using the calendar from mobile a bit too complicated. I decided to create an index file to store the correspondences between short mnemonics that I could add in the description and the full URL of the matching icon.
This index file could be a simple text file with a super-simple syntax.
```
rugby:https://docs.google.com/...
party:https://docs.google.com/...
pool:https://docs.google.com/...
trip:https://docs.google.com/...
ski:https://docs.google.com/...
end
```
I used a semi-colon to separate the mnemonic tag from the URL and the "end" keyword to mark the end of the list (more on this when we will discuss the code in detail).
In this way, I can store my icons on google drive, one drive, Dropbox or any other file-storage service without having to upload them to my own website. The only requirement is that the URL points to a direct download of the file (this is a bit tricky to get using google drive, but you can find a description on Instructables).
Icons themselves are stored as simple BMP files. You can export them from Microsoft Paint (described on instructable) or any other software that can generate BMPs in the monochrome uncompressed format. You just need to check that file size is 446bytes (48x48 bits of data plus a fixed-size header).

## Secure connection

When I talked about "downloading" information from a server or a file-storage service I really over-simplified the process, from the point of view of our poor small device.
The main issue is that you need to download those pieces of information using a secure connection. Connections used to download our calendars, images and an index file for the images are secured using the HTTPS protocol. When you use HTTPS URLs in your browser they behave exactly like "plain" (and insecure) HTTP ones, but what is happening "under the hood" is quite different.
When you use HTTPS the client and server need to exchange encrypted data. In this way, no one can read the data exchanged between them.
Security in HTTPS is based on certificates. A certificate allows a client to trust that the server is who it's claiming to be and that information can be exchanged safely. This is what prevents malicious websites from impersonating legitimate ones and keep the "secure" mark that your browser shows near the URL.
But how can the client know that a certificate is valid and so the server is actually legitimate? Another certificate!
Certificates can point to other certificates in a chain of trust.
At some point, certificate after certificate, you should reach a top-level certificate that everyone on the internet (or, at least, the two parties involved) recognized as trusted. 
You have some of those certificates, belonging to well know certification authorities, on your PC right now.
Usually, embedded devices can't store a large amount of data, and storing all the certificates commonly stored on a PC is not doable, that's why I developed a small Arduino library that can be used to store the certificates needed to access your calendars (see my previous post about it).
Using HTTPS to download your calendars grants that our poor little device could not be "fooled" in downloading a wrong or malicious calendar from a different location. This is probably not needed for such a simple homemade device but can become an issue for commercial products.

## Redirects
Sometimes a web server may not actually host the content you are looking for. When you try to access it with an HTTPS request the server "redirects" you to a different location. This is something that happens very often when we navigate using our browser, but we rarely notice that because the client software is kindly handling that without requiring our attention. For our poor small embedded device, this means opening a new connection, validating a different server etc.

## Conclusion 
Collecting data from the Internet is not as easy as it seems!
As you can see collecting data from the internet is not simple, even for an apparently simple and limited application. Internet protocols are designed to grant security and many times to support clients running on a full-featured PC. This is what makes data access on devices quite complex and challenging considering the small amount of computing and storage resources we have.
It also means decoding complex data, most of the times in textual formats that makes them readable for humans but more complex to parse for computers.
In one of the upcoming articles, I plan to describe a different approach to this same project, involving some server-side services, that could make the implementation on the device much simpler and reliable.
This would make sense if someone plans to create a consumer product based on this idea.



