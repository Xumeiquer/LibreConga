# Protocol

First of all, I'd like to mention something about SSL Pinning.

After capturing the login process I felt like the traffic was not ciphered so I tried to login into the Conga app on another device, obviously, without installing any CA or rooting it. The traffic was the same, well it follows the same pattern. At the end of the day, all the stuff I set up for bypassing the SSL Pinning was useless, well I learned a lot, but nothing worth it for this case.

That said, let's continue with the login protocol.

Once the application is open, it starts sending `ping` messages to the server. Those messages are as following:

```
18 00 00 00 | 02 00 00 00 | 00 00 00 00 | 00 00 84 f2 | 38 34 41 30 | 6e 5d d5 07
```

I did a small research on the Internet and I found the [BadConga project](https://github.com/adrigzr/badconga) where the owner did some reversing on those pings. They are following a pattern described [here](https://github.com/adrigzr/badconga/blob/master/parse.py#L42).

```
Length:    18 00 00 00
ctype:     02 00
flow:      00 00
user id:   00 00 00 00
device id: 00 00 84 f2
Sequence:  38 34 41 30 6e 5d
OpCode:    d5 07
```

It seems that `18 00 00 00` is the size of the data send back and forth. If I convert that into a decimal value it decodes as `24 00 00 00`. 24 is the number of bytes there are in the frame. But, this value is not `24` in decimal, that value is `24000000` so it must be encoded as [little-endian](https://en.wikipedia.org/wiki/Endianness). Decoding `18 00 00 00` as little-endian it makes more sense now.

## Login request

When I entered the credentials and hit login, the application sent soma data to the server. That data was as follows.

```
41 00 00 00 | 02 00 00 00 | 00 00 00 00 | 00 00 ee d7 | 49 34 41 30 | 6e 5d b9 0b | 0a 13 XX XX | XX XX XX XX | XX XX XX XX | XX XX XX XX | XX XX XX XX | XX 12 0f 3f | YY YY YY YY | YY YY YY YY | YY YY YY YY | YY 20 eb 07
```

I can see some pattern here, `41h` is `65d`, which is the exact amount of data that is in the packet. If I try to decode it as before...

```
Legnth:    41 00 00 00
ctype:     02 00
flow:      00 00
user id:   00 00 00 00
device id: 00 00 ee d7
Sequence:  49 34 41 30 6e 5d
OpCode:    b9 0b
Payload:   0a 13 XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX 12 0f 3f YY YY YY YY YY YY YY YY YY YY YY YY YY 20 eb 07
```

Where the bytes marked as XX are part of my account and bytes marked as YY belong to my password.

I'd like to make some notes here. This is not a normal protocol as it could be Form POST or JSON, it seems to be a serialized protocol, but I don't know yet which one. It is weird to build up a new protocol, but there are crazy people around there.

Another thing I noticed. The field _device id_ changes on every ping request, and it could be even normal as no one has login yet. But, once logged in why is the application sending a completely different _device id_? Well, let's see whether it changes once the Conga gets paired.

## Login response

Following my logic, the server response should follow the same protocol so let's dig into the reply from the server.

Once more, I am going to split up the frame into chunks. Some of them have a meaning already, others are new.

```
4d 00 00 00 | 07 01 47 00 | 00 00 00 00 | 00 00 ee d7 | 49 34 41 30 | 6e 5d ba 0b | 08 00 1a 31 | 08 c3 be 28 | 12 20 SS SS | SS SS SS SS | SS SS SS SS | SS SS SS SS | SS SS SS SS | SS SS SS SS | SS SS SS SS | SS SS SS SS | SS SS 18 01 | 28 00 32 05 | 08 eb 07 10 | 03
```

Arranging it a little bit.

```
Length:    4d 00 00 00
ctype:     07 01
flow:      47 00
user id:   00 00 00 00
device id: 00 00 ee d7
Sequence:  49 34 41 30 6e 5d
OpCode:    ba 0b
Payload:   08 00 1a 31 08 c3 be 28 12 20 SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS 18 01 28 00 32 05 08 eb 07 10 03
```

The bytes marked as SS belongs to something that seems to be a session token or similar. It is constructed from hexadecimal characters. All other characters seem to be some kind of context of wrapping data. But I don't have enough information to say something about them.

Note: The _Sequance_ is constant during the session, which means when the application is opened until it is closed. As I said, the _device id_ changes on every request. And, the _op code_ is an operation code. This is what I've got so far.

| Op Code   | Decimal | Meaning              |
| --------- | ------- | -------------------- |
| 0xd5 0x07 | 2005    | Client ping          |
| 0xd6 0x07 | 2006    | Server ping response |
| 0xb9 0x0b | 3001    | Login                |
| 0xba 0x0b | 3002    | Login successfull    |

## Logoff request

## Logoff response
