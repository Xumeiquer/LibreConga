# Pairing

Cecotect provides a bunch of mobile applications. There is like an application per model series. That suggests there are either potentially several protocol implementations or there is one protocol with hardware limitations (depending on the model)

## How Cecoted has designed the pairing between the mobile app and the Conga?

I am using the Conga 5000 app.

- Android: https://play.google.com/store/apps/details?id=es.cecotec.s5090&hl=es_419&gl=US
- iOS: https://apps.apple.com/es/app/conga-5000/id1469506857

The application asks for a lot of permissions, but that is not the point at the moment. Thus, the process requires you to connect the mobile to a Wireless net with Internet access. Hence, the pairing wizard asks you to give a Conga's name. Next, the application brings up the network SSID name so you can pick it up. Next, you can define the password. Finally, the applications do some connections and end pairing the Conga. That pairing allows the Conga to connect to the Wireless network with Internet access.

## Sniffing the pairing process

I configured a Raspberry Pi with a wireless AP so I can monitor all the traffic that goes through the wireless connection.

Once I have that AP configured in the Raspberry Pi. I connect the mobile to that AP then I follow the pairing process.

First, I filter DNS traffic and I can see there is an A query for the domain `eulog.3irobotics.net`. The answer gives a CNAME pointing to `eu.log.3irobotics.net`. That last FQDN has the following IP assigned:

- eu.log.3irobotics.net: type A, class IN, addr 8.211.48.101
- eu.log.3irobotics.net: type A, class IN, addr 47.254.145.60
- eu.log.3irobotics.net: type A, class IN, addr 47.91.87.185
- eu.log.3irobotics.net: type A, class IN, addr 8.211.50.7

Second, I have to see any connections going to one of those IPs. The filter to apply in [Wireshark](https://wireshark.org) is `ip.dst == 47.254.145.60 || ip.dst == 47.91.87.185 || ip.dst == 8.211.50.7 || ip.dst == 8.211.48.101`. That brings up some TCP connections and **TLSv1.2**. I highlighted the TLSv1.2 because that means the connections are encrypted, that is because the application uses HTTPS. Now, I have to see whether it is HTTPS or HSTS. If the app uses HTTPS I can bypass that easily, on the other hand, HSTS is much more difficult, but it can be bypassed anyway.
