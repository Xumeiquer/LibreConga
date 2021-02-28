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

## Protocol Buffers

After some digging, I found the protocol used in the payload or data section is
[Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview). This is a binary protocol designed by [Google](https://google.com).

In terms to use this protocol, you must define how your message looks like and then compile that definition. Thus, you can use it in your applications.

An example from the documentation's side.

```protobuf
message Test1 {
  optional int32 a = 1;
}
```

You create a `Test1` object and assign 150 to the `a` field. You then serialize the message to an output stream. If you were able to examine the encoded you'll see three bytes.

```
08 96 01
```

[Here](https://developers.google.com/protocol-buffers/docs/encoding) is the full documentation regarding Protocol Buffers.

## Reversing Protocol Buffers

Protocols Buffers are sent over the network as a binary stream of data so I need something to read that data and convert it into a readable object, something similar to JSON. There is a tool written in Python called [blackboxprotobuf](https://github.com/nccgroup/blackboxprotobuf) that can help me on this matter.

Blackboxprotobuf is really easy to use. It exposes a method that requires a binary string as an argument as input. The method returns back a Python set, which can be treated as JSON object.

Protocol buffers are reversed as `key: value`. But the key is not auto-explanatory, instead, the `key` is a number that is used when the data is defined just before it gets compiled. However, I can use whatever key I want in my data model definition as long as I assign the same **field number**.

### Login request

The login data was encoded in Protocol Buffers so using Blackboxprotobuf I can recover what the message was sent off to the server.

Just to catch up, the login data was defined like:

```
0a 13 XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX 12 0f 3f YY YY YY YY YY YY YY YY YY YY YY YY YY 20 eb 07
```

The `XX` bytes are for the username and the `YY` bytes are the ones for the password. But there is more information not visible in plain text.

After running the Blackboxprotocol I could see another field.

```json
{
    '1': b'email@addres.com',
    '2': b'<password>',
    '4': 1003
}
```

From here I know that the Protocol buffers model for the login request should be something similar to this:

```protobuf
message LoginRequest {
  required string email = 1;
  required string password = 2;
  required int32 unknown = 4;
}
```

### Login responses

Following the same approach, I decoded the data received as a response from the login request, and as well as the login request I was able to see more fields.

The data returned from the server.

```
08 00 1a 31 08 c3 be 28 12 20 SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS SS 18 01 28 00 32 05 08 eb 07 10 03
```

`SS` bytes are the ones for the session token I could identify as they were in plain text.

This is what Blackboxprotobuf could identify.

```json
{
    '1': 0,
    '3': {
        '1': UUUUUU,
        '2': b'SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS',
        '3': 1,
        '5': 0,
        '6': {
            '1': 1003,
            '2': 3
        }
    }
}
```

Clearly, the data was hiding more info than expected. I classified `UU` as the `user ID`, but I don't know yet.

<!-- prettier-ignore-start -->

!!! note
    The following requests and responses had the value `UserID` in the header populated with the value **UUUUUU**. Thus, field  `3.1` is the user identification.

<!-- prettier-ignore-end -->

Now what I can see is that the login response data model is something like.

```protobuf
message LoginResponse {
    message Unknown {
        required int32 unknown1 = 1;
        required int32 unknown2 = 2;
    }

    message Msg {
        required int32 user_id = 1;
        required string session = 2;
        required int32 unknown = 3;
        required int32 unknown1 = 5;
        required Unknown unknown2 = 6;
    }

    required int32 unknown = 1;
    required Msg msg = 3;
}
```

## Subsequent request/responses

Now, I know how to decode the Protocol Buffers so reading the data and parsing as Protocol Buffers is not an issue anymore. The problem is to identify all the field meaning. This is usually a try and failure and sees what changes on each request by doing _A_ or _B_.

Just after the login success response from the server, the application did another call to the server with the following data.

```json
{
    '1': b'SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS',
    '2': 1,
    '3': 1003,
    '4': 1003,
    '5': b'1.1.21'
}
```

Fron this request I got that the application was using the session returned by the server on the login success. The application also sends some other values that I don't understand, but there is one `'5': b'1.1.21'` which is the application version. On the other hand, the interesting thing here is the command `opcode` used to send that amount of data. The opcode is **0x7d1**. This code seems to be a client identification or something similar.

I didn't receive any response, instead, the app did another request.

```json
{
    '1': UUUUUU,
    '2': b'SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS',
    '3': 23
}
```

This time the `opcode` was **0xbcb** and it seems to be a kind of ping or keepalive.

Another `opcode` sent to the server was `0xbe5`. As per the information sent it seems to be just a remainder for the application version.

```json
{
    '1': 1,
    '2': 1003,
    '3': b'1.1.21'
}
```
