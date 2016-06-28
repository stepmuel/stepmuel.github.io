---
layout: post
title: "Why Apple Might Not Like the Headphone Jack"
date: 2016-06-28 15:18:21 +0200
tags: apple
---

There are rumors about Apple ditching the headphone jack on future iPhones. Will they do it? Why? Nobody knows! But speculation is fun. Let me contribute an argument on how a new port could potentially improve the current situation not for headphones, but for headsets. 

# The second ring

The regular headphone jack we all know and love offers 3 pins: the **t**ip (left audio), one **r**ing (right audio) and the **s**leeve (ground). This configuration is called TRS and has been used to transmit stereo audio signals to headphones or between audio devices for decades. The connector is widely used, reliable and the only drawback I can think of is that they create short circuits while plugging and unplugging. 

The situation gets a bit more complicated when adding a second ring, which is needed to support microphone input and remote functionality. Wikipedia [states](https://en.wikipedia.org/wiki/Phone_connector_(audio)#PDAs_.26_mobile_phones):

> There is no recognised standard for TRRS connectors or compatibility with three conductor TRS. The four conductors of a TRRS connector are assigned to different purposes by different manufacturers. Any 3.5 mm plug can be plugged mechanically into any socket, but many combinations are electrically incompatible.

# User input

In addition to providing remote buttons and microphone input, the headset port has been used to provide simple data input without needing certification by Apple. A notable example is the [square reader](https://squareup.com/reader), used to read credit cards with an iPhone. I developed a low-cost UV sensor for smartphones which used the same interface. The fundamental principle is to mimic a microphone and restore the transmitted data from the audio stream available to the app. 

In order for the microphone to work, Apple puts a bias voltage of around 2.7 V between the ground and the microphone pin. The microphone will then convert audio waves to small variations to that bias voltage. There is a simple pairing process to detect whether a plugged device has a microphone. Simply put, the phone measures the resistance between the microphone pin and the ground pin. For TRS jacks, both pins are connected to the sleeve, shorting them to ground. If the resistance is within a certain range, the phone will assume that there is a microphone present and use it as input device. If not, it will turn off the bias voltage and thread the device as output only. 

And yet again, there is no official standard to follow. The timing for the pairing process, the bias voltage and the resistance ranges are different, even among Apple products. Some Android phones don't have an upper resistance limit and just keep the bias voltage on. 

Once the headset is detected, it can also be used as a remote. The play/pause button simply shorts the microphone pin to ground for as long as it is pressed, effectively muting the microphone signal. The volume buttons will add some custom modulation to the bias signal which I haven't fully analyzed, and neither did many others since volume buttons are mostly lacking from cheap 3rd party headsets. 

While the audio jack is wonderful for transmitting audio, using it bidirectionally might be a smart hack, but has many flaws and lacks standardization. 

# Buying experience

Apple moving to a different audio connector would very likely make Android and iPhone headphones incompatible. In a recent [episode](https://www.relay.fm/upgrade/95) of the Upgrade podcast, [Myke Hurley](https://twitter.com/imyke) mentioned (at 53:40) that this would result in a worse buying experience, since customers would have to check for compatibility. 

While this sounds plausible for regular headphones, it might be the opposite for headsets with microphone and remote buttons. Most headsets are already sold in separate versions for Android and iPhone. Even if they are supposed to work with iPhones, they [might not work well](http://thewirecutter.com/2015/03/a-word-on-the-iphone-6-iphone-6-plus-and-third-party-headphones/). As someone who often uses the volume buttons to adjustment for ambient noise while walking outside, picking a pair of new headphones requires a lot of research. Being able to rely on "made for iPhone" logos would make things easier, at least for me. 

Apple already offers "made for iPhone" to headphone manufacturers. As far as I understand, Apple usually provides the microphone and a chip to make the remote work. However, since the electrical characteristics and the remote protocol can easily be reverse engineered, almost everyone is avoiding the licensing costs and just uses their custom made solution. 

Apple might have many support requests caused by unreliable or incompatible headsets. Requiring manufacturer to have their headsets certified would allow Apple to control the quality of those 3rd party accessories. If Apple sees bad headsets as a problem, moving to a proprietary port seems to be something Apple would do, and might be the only way to get rid of them. 

(Please note that I'm simply adding one more argument to the headphone jack debate without the intention of invalidating any previous arguments made by others.)


