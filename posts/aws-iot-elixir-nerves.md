# AWS IoT, Elixir, and Nerves: A Crash Course

Brief intro through Elixir and Nerves and high level outline of blog, we will
talk about:

  * what AWS IoT and NervesHub actually do
  * security comminality between the two (client side certificates)
  * moving forward

## AWS IoT

Brief description of AWS IoT:

AWS IoT is Amazon's core IoT offering. This service:

  * acts as a communication hub for IoT devices.
  * manages actions taken (lambdas, etc.) in response to messages published on
    MQTT topics
  * manages things, thing types, and thing groups
  * manages security items like device certificates, policies, and CA
    certificates
  * Leverages client side certificates
  * Use Tortoise to talk to AWS IoT

## NervesHub

Brief description NervesHub:

NervesHub is written in Elixir to facilitate secure and dynamic OTA firmware
updates for embedded devices. This service:

  * manages devices, firmware, and deploys
  * firmware ttls
  * complex firmware resolution (tags and complex version resolution)
  * Supports real time updates
  * Leverages client side certificates

## Client Side Certificatees

Big positive security implications.

SSL stuff: Verify peer, partial chain verification

Cert and key on device, verifies identity of server. Server verifies identity
of device.

Mitigates MITM attacks.

Facilitates Just-in-Time Provisioning.

## Just-in-Time Provisioning

Solves the problem of oh I need to make a lot of these fast.

AWS leverages configuration tied to CAs to setup JIT provisioned devices.

To get started quick NervesHub can create and sign certs for you, but eventually
you will probably want to switch to a custom CA.

A custom CA gives you the control you need to manage a fleet of IoT devices.

NervesHub requires:

  * the device-identifier in the CommonName
  * to have created the device already

## Moving Forward

Ensure CI is setup to build and publish FW. Whether you just deploy to
production devices or not depends on context.

Manufacturing a device requires certs and a base firmware.

Once eligible devices will be made aware of new firmware and grab it. Devices
pull down firmware and retain their certs.

Take actions based on published messages.

Roll out firmware to groups of devices.

## Closing Remarks

IoT development can seem like a lot but general software engineering best
practices still apply.

AWS IoT, Elixir, and Nerves facilitate agile development of robust, scalable
software.
