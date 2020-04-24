+++
date = "2017-12-13"
title = "BAAAHS Lighting Controller Development"
description = "Details on development of BAAAHS 2nd-gen light control hardware debuted at Burning Man 2017"
tags = [ "baaahs", "ansible", "infrastructure as code"]
aliases = [ "/blog/baaahs-lighting-controller-development/", "/blurbs/baaahs-lighting-controller-development/"]
+++

The Big Ass Amazing Awesome Homosexual Sheep ([BAAAHS](http://www.baaahs.org)) is a well known fuzzy entity in
the Bay Area queer maker and burner space. BAAAHS garners attention with its peacock-like
bedazzling light shows, light beacons, earth-shattering beats, and two-story
dance and lounge spaces. Oh. And a pool slide. I can't forget that pool slide!

![image](/img/2017-baaahs-lighting-controller/baaahs-0.jpg)

2016 was my first year burning with BAAAHS. In 2016 we set out to complete an
ambitious number of new structural changes to Pearl (BAAAHS' other gendered daytime
name). We decided early in 2017 that projects would be kept as light as possible, perhaps
with an eye towards efficiency. I took the opportunity to make enhancements to our
lighting and light control systems.

The existing system was simple; a laptop running code writing DMX signals to
controllers that drive lights. The system mostly worked, but it was _fussy_. The system
needed a few hardware modernizations, documented process and infrastructure code,
and training on how to administer and use the system. We set out to achieve
all this for 2017.

# Existing Architecture #

The existing system consisted of a non-SSD-based netbook, a router/switch with
underpowered transmission capabilities, a partially-operational UPS (functioning as a strip only), a DMX to ethernet
bridge, and a slew of random cabling. All of these parts lived and mingled with
all the playa dust in an open milk crate.

![image](/img/2017-baaahs-lighting-controller/old_and_new.jpg)

Driving over a hundred lighting panels and forward facing sharpies, the system
mostly works as expected for its users. On playa having a system that *mostly*
works is pretty much the bar to aim for. There was,
however, room for improvement in some low (and high) hanging fruit...

### Power Resiliency ###
The UPS' battery was years old and had suffered through the
worst of power conditions. It was dead. The power supplied by an auxiliary
generator was _mostly_ stable. When it wasn't stable it would cause the laptop's
network interfaces to go down. This, in turn, would cause chaos between the OLA
daemon driving the lights via the networked DMX bridge and the BAAAHS light
server service. Bounced power, even if for a second, meant
rolling a life-endangering, exhaust overcoming save and crawling in the
jefferies tubes to restart things.

![image](/img/2017-baaahs-lighting-controller/baaahs-2.jpg)

### Hardware Failure Mitigation ###

The hardware we had was already operating with multiple _workable_ failures.
Any additional, single component failure could render us in bad shape. For example,
the critical server and services ran from a 4-year-old, platter-based netbook.

### Procurement and implementation docs ###

The hardware we had was old, it had gone through many undocumented/ad-hoc
configuration changes over time, and many parts were no longer procurable.
In most cases it was not precisely clear what to re-order and how to
rebuild it should we need to do so.

### Construction and User Training ###

Only a few individuals understood how the server lit up lights. Failures
generally meant unsuccessfully chasing down the few BAAAHS shepherds in the know.

# New Architecture #

The new architecture would be headless and based on the Raspberry PI 3. The PI 3's armv8
multi-core processor was more than enough for what we needed (perhaps close to as
powerful as the generations-old Atom in the netbook it was to replace).

![image](/img/2017-baaahs-lighting-controller/internal_controller-0.jpg)

Other than migrating to the RPI3, we aimed for the following architecture and
process improvements:

- Single unit, industrial-grade hardware meant to replace the many individual components
- Purchase list of commodity hardware necessary to build a light server
- Full infrastructure as code required to produce a working light server
- Documentation and user training of all systems
- Development of additional services necessary to deliver Touch OSC layouts
to mobile users
- Internal, wireless networking
- General higher degree of system resiliency, auto healing

### Single Unit ###

Single units were important because it would allow us to have fully functioning
backups in a small amount of space that can easily be mounted on or near to the lights
and control boxes. This would be particularly useful for off-playa events wherever
infrastructure was used.

Achieving single unit meant testing expensive USB <-->
DMX interfaces, high-power USB adapters with the proper chipsets to play nice
with hostapd in Linux (that which makes an access point), internal power backup
"UPS" HATs, and cases.

Most of the effort for Single Unit production meant painstakingly scouring the web
for products, digging for their hardware specifications, and a lot of Amazon returns.
It's difficult to know exactly what, say, the power draw on the PI will be from a USB
<--> DMX bridge where all other competitors warn of high amperage pull and what not.

### New Architecture: Parts List ###

It seems obvious, but as people come and go from the community, or take a year
off, their knowledge of the exact parts used and their function is too. This
adds potentially significant effort after the fact, in the case of failure, or
in the general case another unit is required.

- [Raspberry PI 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
- [Raspberry Pi 2/3 Copper Heat Sink Heatsink](https://www.amazon.com/gp/product/B01GM9EYQ8)
- [Duinocases Industrial, Metal Enclosure](http://www.duinocases.com/store/raspberry-pi-enclosures/duinocase-b-enclosure-for-the-raspberry-pi-b/)
- [UPS PIco HV3.0A 450 mAh Stack](http://www.pimodulescart.com/shop/item.aspx?itemid=30)
- [SanDisk Extreme 16GB UHS-I/U3 Micro SDHC Memory Card](https://www.amazon.com/gp/product/B00M55BX3G)
- [DMXking ultraDMX Micro USB DMX adapter/dongle](https://www.amazon.com/gp/product/B00T8OKM98/)
- [CanaKit 5V 2.5A Raspberry Pi 3 Power Supply / Adapter / Charger (UL Listed)](https://www.amazon.com/CanaKit-Raspberry-Supply-Adapter-Charger/dp/B00MARDJZ4/)
- [Panda Wireless PAU06 300Mbps N USB 802.11 Adapter](https://www.amazon.com/Panda-Wireless-PAU06-300Mbps-Adapter/dp/B00JDVRCI0)
- [Kinobo - USB 2.0 Mini Microphone "Makio" Mic](https://www.amazon.com/gp/product/B00IR8R7WQ)

### Infrastructure as Code ###

An [ansible playbook](https://www.github.com/baaahs/light-server-infrastructure) was created to provide the
self-documented, repeatable process of constructing a server out of the
aforementioned list of commodity hardware. A server is fully functional
and ready to go after execution of the playbook.

The playbook executes a number of roles aimed at:

- The installation of all custom BAAAHS services (ola layout server, lights server, custom scripts, beat detection)
- Endurance optimizations meant to keep unnecessary writes from the SD card (for `/tmp`, `/var/log`, etc...)
- Wireless network creation and configuration
- Custom OLA installations
- Performance optimizations such as SD overclocking and reduction or elimination of unnecessary service usage
- UPS pico-specific (the PI HAT UPS) setup

### Documentation ###

In addition to the documentation within the infrastructure ,code bases are
additional design, usage, and training documents and presentation materials.
Prior the playa this year, many BAAAHS shepherds underwent training on writing
lights server shows, construction, technical setup, troubleshooting and other
training workshops.

View and overall presentation of all the BAAAHS lighting systems [here](/pdf/2017-baaahs-lighting-controller/baaahs-lights.pdf).

### Additional Services ###

Additional services were created to assist BAAAHS shepherds in troubleshooting
and diagnosing problems with the lighting systems. This primarily included
custom scripts, messages, and on-device documentation of the systems and feedback
such as temperature, load, service status, etc...

On top of that, 2017 was the first year the controlling systems used
real-time environment feedback (in form of sound analysis) to provide
automated adjustments to lights from DJ playback. Woo hoo!

### Internal Networking ###

The RPI3 has a few network interfaces used for different things:

- An external, high-powered USB 802.11 adapter used to create access points
- An internal 802.11 adapter used to connect to seeded list of *other* access
points (home networks, mobile phone, and other LTE-based hotspots)
- An internal 100Mbps ethernet adapter used as either an admin interface
with a crossover cable or a dynamic port for an existing LAN.

# Looking forward #

![image](/img/2017-baaahs-lighting-controller/baaahs-1.jpg)

The hardware re-design, process re-design, documentation, and training
all contributed to a much more smooth and rapid deployment and operation of BAAAHS on the playa this
year. I can't wait to see what projects and opportunities arise in 2018.
I'm hoping to help make fire happen! If any of this sounds interesting reach out,
inquire, and join us in 2018.
