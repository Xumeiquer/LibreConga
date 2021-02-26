# SSL Pinning

There are several techniques to bypass SSL Pinning, some are easier than others. I am going to use an old Android tablet that runs Android KitKat 4.4. I think that will help me a little bit because that version does not implement [Network Security Config](https://developer.android.com/training/articles/security-config) which was added in Android 7.

## Adding custom CA

I am going to add my custom CA so using [BurpSuite](https://portswigger.net/burp) I'll be able to decrypt the TLS traffic and see it in plain text.

As I am running Android 4.4 it should be enough to add the CA in the Keystore. I configured Burp to listen on any interface so when navigating to http://burp from the Android browser I'll be able to install the CA.

## Setting up RPi

When examining the network traffic and doing some changes it is worth avoiding doing it on your own machine, otherwise, you may get block without an Internet connection. Thus, I configured a [Raspberry Pi](https://www.raspberrypi.org/) 4 with [Raspbian](https://www.raspbian.org/) so I can handle the traffic modifications without any issues on my own machine.

The setup is quite easy. First, I'll need an Access Point to connect the tablet to. Second, I need the AP software (hostapd) and a DNS server so I can handle the resolutions at my wish.

In terms to configure the DNS responses well an catch them I sniffed the traffic to see what connections was doing the Conga app. At this point, I realized that when opening the application even without being authenticated, the application was doing another connection. This time to `cecotec.fas.3irobotix.net:4020`. The data send it to the server was like:

```
180000000200000000000000000081de383441306e5dd507
180000000200000000000000000082e3383441306e5dd507
180000000200000000000000000083ed383441306e5dd507
180000000200000000000000000084f2383441306e5dd507
180000000200000000000000000085fc383441306e5dd507
```

I did some tests and I could see most data is static, but some changes.

1800000002000000000000000000**81de38**3441306e5dd507

That highlighted bit changes across requests and every time the application is opened.

In particular 1800000002000000000000000000**8**1de**38**3441306e5dd507 changes every time the applications is opened, however, 18000000020000000000000000008**1de**383441306e5dd507 changes on every request. Digging a little bit there is a nibble that its count is sequential, 18000000020000000000000000008**1**de383441306e5dd507. Thus, this can be a kind of request counter or keepalive.

As the data seems to be quite different from the usual HTTP protocol I set up a [MitM relay](https://github.com/jrmdev/mitm_relay) so I can grab the traffic send using a different protocol than the HTTP.

Doing a quick overview of the data seems to be something. Splitting it up in chunks of 4 bytes long I got 6 blocks.

```
18 00 00 00 | 02 00 00 00 | 00 00 00 00 | 00 00 84 f2 | 38 34 41 30 | 6e 5d d5 07
```

## Setting up BurpSuite

In BurpSuite I didn't do a lot, basically, I configure a proxy for all interfaces listening at the same port a used in the MitM `proxy` flag.

## SSL Pinning

So far so good, but where is the SSL. Well, I am guessing that the BurpSuite CA installed through the browser did the job, as I can see that traffic or it might be in plain text I am not fully sure at this point as I didn't sniff the traffic before installing the CA.

What I did to see whether the application uses SSL Pinning was disassembling it. Using [ApkTool](https://ibotpeaches.github.io/Apktool/) I could see in the pseudocode that the Cecotec application uses [okhttp](https://square.github.io/okhttp/). That is because they are implementing the SSL Pinning and the BurpSuite CA installed may not be enough.

The way I bypass the SSL Pinning is by using [Objection](https://github.com/sensepost/objection). That was the first time I use that tool and what the hell, that is fucking awesome. It can do a lot of crazy things and one of them is bypassing the SSL Pinning. I am not going to explain more as the documentation is really nice and the tool is really easy to use.
