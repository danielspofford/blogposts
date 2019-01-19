# AWS IoT Core, Elixir, and Nerves: A Crash Course

Find out what is happening at the intersection of an IoT device management and
pubsub service, and a fault tolerant language leveraging a powerful platform.

--

## Introductions

AWS IoT Core is a managed cloud service that facilitates managing devices,
securely communicating to and from them, and taking action based on their
messages.

Elixir is a functional language well suited for maintainable, low-latency, and
fault-tolerant systems. It runs on the Erlang VM which allows it to leverage
decades of production battle-testing and libraries.

Nerves is the combination of a platform, a framework, and tooling, where the
output is the ability to "craft and deploy bulletproof embedded software in
Elixir".

NervesHub is written in Elixir by the Nerves core team, though both are open
source and contributed to by many others as well. It facilitate secure and
dynamic OTA firmware updates for embedded devices. Keep in mind it is a managed
service, but it can be managed privately as well, and there is a host of
Elixir libraries under the umbrella of Nerves/NervesHub that one may leverage on
an IoT device.

## General Management

### AWS IoT Core

AWS IoT Core manages the CRUD of many important pieces of the IoT puzzle
including but not limited to: Things, Types, Thing Groups, Billing Groups, and
Jobs. The most fundamental piece is a Thing, which is the AWS term for some IoT
device. A Thing could be a doorbell, a camera, a thermometer, or in reality,
anything. These Things allow you to manage and take certain actions even when a
Thing is not currently online. For example, leveraging Device Shadows allows you
to read the last known state of a Thing and even edit it, so that the next time
that Thing comes online it could respect those changes.

Types are just categories of Things, Thing Groups are as they sound but
critically they allow you to attach policies and configure logging options at
the Group instead of Thing level. Billing Groups are similar to Thing Groups,
but can be associated with tags which facilitates categorization and tracking of
IoT related costs. Finally, Jobs are essentially a remote command that will be
interpretted by one or many Things, but with bells and whistles. Jobs can be
scheduled, applied to Thing Groups, or rolled out in stages. All throughout a
Thing starting, working on, and finishing a Job, it will report its progress to
AWS IoT.

### NervesHub

NervesHub manages the CRUD of many important pieces of the secure over-the-air
(OTA) firmware update puzzle including but not limited to: Products, Devices,
Firmwares, and Deployments. At the time of writing, a Product is a name that
acts as grouping for Devices, Firmwares, and Deployments. A Device in NervesHub
is conceptually analagous to a Thing in AWS IoT. It has a name, is associated
with a device key and certificate (though NervesHub is only ever privy to the
certificate), and it can be assigned tags. A Firmware is a signed chunk of
binary information associated with some metadata like version, VCS identifier,
author, time-to-live, etc.

Deployments are where the rubber meets the road. They represent a resource that
can be assigned a Firmware, tags, and version requirements among other things.
This facilitates very powerful resolution of eligible firmwares for a Device.
Through version requirements and tags one can dictate a complex multi-step
upgrade path for Devices.

## Communication

### AWS IoT Core

The communication protocol of choice between Things and Iot AWS Core is MQTT due
to its low bandwidth overhead, small code footprint on clients, and topic based
messaging. It is worth noting that while MQTT has 3 QoS levels (0, 1, 2), AWS
IoT only supports 0 and 1. Remember that Things can not only publish messsages
to topics to be processed in the cloud, but also can subscribe to topics so that
information can be pushed down to them as well.

To tie off that functionality are Actions: a convenient way to react to messages
published to topics. This is not an inflexible configuration either, AWS IoT
Core allows you to leverage MQTT topic wild cards like `+` and `#` to specify
multiple topics at once. When an action is triggered you can take predefined
actions like writing to a dynamo table, adjusting a CloudWatch alarm, or a whole
host of other actions, but the escape hatch being invoking a lambda which allows
you to do whatever you want.

### NervesHub

NervesHub and AWS IoT do not need to talk to each other, but your device does
need to talk to NervesHub. For this you can leverage the Elixir library
aptly named `NervesHub`. This will allow your device to maintain a websocket
connection to the NervesHub service to allow real-time pushes of firmware and
facilitate useful metadata like whether a device is currently online or not
(this uses the Phoenix framework's Presence feature).

This always-on connection may be restrictive contextually if you have strict
power or bandwidth concerns. In which case you simply do not leverage the
real-time features of the NervesHub service and instead opt to poll at your
leisure. In order for your device running Elixir to speak MQTT to AWS IoT, Martin
Gausby's Tortoise library will do the trick.

## Security

### AWS IoT Core

AWS IoT leverages public key cryptography to ensure secure communications to and
from Things. When a Thing initiates communication with AWS the following occurs:

  * A TLS handshake.
  * The server presents its certificate to the Thing.
  * The Thing uses a root AWS cert on itself to verify the Server's certificate.
  * The thing presents its device certificate to the Server along with a hash of
    the conversation so far signed by its device key.
  * The Server uses the Things public key (which was inside the Things device
    certificate) to verify the signature on the hash the Thing provided.

At this point the Server and the Thing can trust that their communication is
secure. Any further security beyond the boundary of the AWS cloud is managed by
the combination of AWS IoT Policies and traditional IAM Policies. Through these
one can dictate the particulars regarding a Things ability to connect,
subscribe, publish, receive, etc. The same control is allowed over Actions and
the services they interact with.

However, before the aforementioned security dance can occur successfully, there
are some manual steps that must be taken within AWS IoT Core for each device who
wishes to participate:

  * A Thing has been created for that device.
  * A Certificate has been created and associated with that Thing.
  * An AWS IoT Policy has been created and associated with that Certificate.

This is fine in development and maybe in a beta-esque phase, but you will
quickly find out that this is kid mode and as the number of Things scale it will
become untenable to provision devices in this manner.

At the application level the previous section mentioned the Thing would leverage
the Tortoise library. Tortoise accepts a variety of options but worth of note is
what the `:server` configuration option passed to a
`Tortoise.Supervisor.start_child/1` call may look like:

```elixir
{
  Tortoise.Transport.SSL,
  alpn_advertised_protocols: ["x-amzn-mqtt-ca"],
  cacerts: ca_certs,
  cert: cert,
  host: Keyword.fetch!(tortoise_opts, :host),
  key: {:ECPrivateKey, key},
  partial_chain: &partial_chain/1,
  port: 443,
  server_name_indication: Keyword.fetch!(tortoise_opts, :server_name_indication),
  verify: :verify_peer,
  versions: [:"tlsv1.2"]
}
```

The `partial_chain/1` function resolves successfully against Amazon root CAs.
Keep in mind when connecting to AWS IoT to:

  * provide the device certificate and key.
  * Provide a set of CA certificates including:
    * Amazon's root CA(s).
    * the CA who signed the device certificate which may or may not be Amazon's
      root CA.

All of that done successfully, a device may connect and publish/subscribe to its
heart's content. Or at least until it runs into an AWS IoT Policy saying
otherwise.

### NervesHub

NervesHub leverages the same security story as AWS IoT: public key cryptography
to ensure secure communications to and from Devices. It also has a similar
pattern of allowing you to generate device certificates and keys using a
NervesHub CA. Keep in mind this certificate and key pair should be the same pair
as is recognized by AWS IoT. When using the NervesHub library to talk to the
NervesHub service it leverages the fact that the device is running on the Nerves
platform to know where the device certificate and key are so unless you go off
the ranch authentication is handled for you.

## Just-in-Time Provisioning

Just-in-Time Provisioning (JITP) solves the problem of "Oh! I need to provision
a lot of these..." so it is what I like to call a good problem, and it even has
a solution so maybe it's the best problem.

### AWS IoT Core

Recall the security section above, the manual steps I mentioned can be
circumvented by configuring JITP. This is done by:

  * registering a custom CA certificate.
  * configuring that CA:
    * to enable automatic registration.
    * with a provisioning template.

Once this is done, the device certificate and key on a device can be created and
signed by the custom CA in your control. The device will also need the custom
CA's public certificate on it as well, this will be presented to AWS IoT during
the authentication steps we outlined previously. Keep in mind the only thing
within AWS at the moment with respect to your device is the CA certificate that
you upload once. Going forward when an unprovisioned device attempts to connect
to AWS IoT the following occurs automatically:

  * Register a Certificate and set its status to `PENDING_ACTIVE`.
  * Create a Thing.
  * Create a Policy.
  * Attach the Policy to the Certificate.
  * Attach the Certificate to the Thing.
  * Update the Certificate status to `ACTIVE`.

Some of the values used for the resources created in that process are influenced
by the provisioning template we configured against the relevant custom CA. These
values are limited to what can be extracted from the subject field of the
certificate of the device being provisioned. At this point the device has become
a Thing and it can communicate with AWS IoT.

### NervesHub

To get started quick NervesHub can create a certificate and key for you signed
by a NervesHub CA, but eventually you will probably want to switch to a custom
CA. This does a couple things for you:

  * Eases manufacturing.
  * You retain full control over the trust chain.
  * You can use the same CA for multiple things where it makes sense.
  * You can leverage a CA external to NervesHub, be it your own custom CA, or a
    well known one like Amazon.

In order to setup JITP with NervesHub one must:

  * register a custom CA certificate.
  * ensure device certificates have an Authority Key Identifier pointing at the
    custom CA certificate.

Going forward when an unprovisioned device attempts to connect to the NervesHub
service the following will happen:

  * A Device will be created.
  * A representation of the device certificate will be persisted.

At this point the device has become a Device and it can communicate with
NervesHub.


## Closing Thoughts

IoT software engineering and devops can seem like a lot:

  * Custom hardware means circuits and wires.
  * Embedded devices may mean power or bandwidth constraints.
  * Low-level/niche context means fewer resources.
  * Side effects of pushing firmware to IoT devices may include
    am-I-going-to-brick-the-world induced anxiety.

To be fair, it is a lot! However, there is no need to fret.

Nerves allows one to start writing firmware quickly. Effortlessly transitioning
from no hardware, to super early dev boards, and finally to production quality
devices all while abstracting you to the high level land of Elixir when you want
it but also allowing you to reach down and mess with the metal when you need it.

Writing firmware in Elixir allows you to leverage high level abstractions, the
fault tolerance and low-latency inherent to OTP and the Erlang VM, and the
efforts of the Elixir maintainers to make the development experience as quick
and productive as possible. NervesHub makes it trivial to manage the full
firmware lifecycle from design to: implementation, upload to a centralized point
of access and management, and finally to complex and dynamic distribution of
that firmware to devices.

Utilizing AWS managed service, IoT Core, lets you scale to billions of devices
and trillions of messages effortlessly, and react to messages from your devices
in powerful ways conveniently leveraging pre-built actions into other AWS
services or reaching out through the invocation of a lambda to a more compelx
series of reactions including those external to Amazon.

In the end, use general software engineering best practices and know that with
AWS IoT Core, Elixir, and Nerves it can be easy to execute agile development of
robust, scalable software.
