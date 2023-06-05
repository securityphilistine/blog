---
title: "Upgrading Lenovo Thinkcenter Tiny machines with Mellanox ConnectX-3 cards"
date: 2023-06-01T12:02:24+01:00
draft: false
tags:
 - Homelab
 - Lenovo
 - Small Form Factor
 - Networking
---

## Introduction

In my homelab I have been having a lot of fun with setting up a small Lenovo Thinkcenter tiny cluster,
and experimenting with Intel AMT and bare metal provisioning systems (coming to this blog soon).<br> 
With the current prices for Mellanox 10Gb and 40Gb cards I have also been getting into higher networking
speeds. At the moment only my ATX machines run with 10/40Gb networking and I mostly use this for fast 
remote file shares and for things like [LANCache](https://lancache.net/).
<br> 
Since then, the 1Gb onboard NICs on the Lenovo Thinkcenter Tiny machines have seemed really sluggish 
and I've been less inclined to host certain services on them. So when I saw Wendel's video on hacky methods of 
breaking device limitations on SFF devices, my mind started racing with terrible terrible ideas.

<div class="frame">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/M9TcL9aY004" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## Networking expansion possibilities for Lenovo Thinkcenter Tiny machines

I have three types of Lenovo Tiny machines in my cluster, I have some M900 machines, some M910q's and an M80q. 
Lenovo offers a few expandability options for these machines, such as optional ports for DisplayPort, USB or even Serial. 
There really isn't much in terms of good network expandability, and certainly nothing in the realm of 10Gb and
beyond. Alternatively you can purchase USB to ethernet devices, some of these even promising speeds as fast as 5Gb, but
reading up on them it seems that it is unlikely that they would be able to reliably hit that. 

### Converting an M.2 NVME to a PCIe slot

This is very viable option for many devices, especially for SFF devices like these tiny Lenovo machines. Nearly
all devices have an M.2 NVME slot and many come with a single SATA port the can be used for primary storage instead. 
Additionally there are lots of options for hardware for such an expansion. The rising demand of bitcoin mining
has causes a surge in devices for maximising PCIe lane you can get from a machine.<br> 

I decided I would run a simple test to see whether I could convert my M.2 slot to a 10Gb network card on one of my tiny machines. 
I purchased an [ADT-Link R43SF 25CM](https://www.amazon.co.uk/dp/B07XMK7ZCR?psc=1&ref=ppx_yo2ov_dt_b_product_details), and attached it
to the M.2 slot of my M80q machine and inserted a basic [TP-Link 10G-BaseT](https://www.amazon.co.uk/gp/product/B08DVGMKWP/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1)
PCIe card in. I turned the machine on and there was no new ethernet device, it needed power. I used an [Aukson P52 USB to SATA](https://www.amazon.co.uk/dp/B0752D5MCL?psc=1&ref=ppx_yo2ov_dt_b_product_details)
cable to give the ADT-Link adapter power. It worked and my machine now has a 10g ethernet port!<br>

<p float="left">
    <img src="/images/lenovo-mellanox-upgrade/adt-link-device.jpg" width="200" />
    <img src="/images/lenovo-mellanox-upgrade/tcp-link-10g-card.jpg" width="200" />
    <img src="/images/lenovo-mellanox-upgrade/usb-sata-power.jpg" width="200" />
</p>

### Connecting Mellanox networking cards 

I had some success, but being greedy, I wanted more. Could I get a 10Gb Mellanox card running here? 
Could I get a 40Gb ConnectX-3 running? 

Whilst the ADT-Link adapter slot can support up to an x16 card, it's still limited by the bandwidth of the M.2 slot.
For all of the Lenovo devices I own, the M.2 slot is a PCIe Gen3 x4 lane slot. This might not seem like a lot, but _in theory_
its maximum bandwidth would be 4GB/s or 32Gb/s. There would be overheads from the networking stack that would probably 
reduce this speed but getting speeds in that region would greatly enhance what these tiny machines could do!

Excitedly I plugged a Mellanox 40g MCX354A-FCBT card in and turned the machine on, it wasn't showing up with `lspci` but I was confident
the card was functional. After a lot of troubleshooting, I realised that most Mellanox cards draw 12 volts for power. The
USB to SATA power cable that I had, whilst SATA connectors _have_ a 12V line, would only ever be able to provide the 5V maximum from USB. 

![Layout of Mellanox card](/images/lenovo-mellanox-upgrade/pcie-cage-inside.jpg)

### Scuffed power solutions

For a quick test, I attached the SATA power connector of the ADT-Link adapter to one of the SATA power cables of my NAS in a very unsafe
balancing act. The card was now functional! <br> 

Obviously using a SATA cable from an already running machine is a terrible idea, so lets do something equally as scuffed instead. 
My idea was to use an old spare Corsair 750W PSU I had laying around to provide 12V SATA power to _all_ my Lenovo Thinkcenter machines. 

To use a power supply without attaching it to a motherboard, you need to use the [Paperclip trick](https://aphnetworks.com/tutorials/psu_paperclip_trick).
I decided to use a proper [pin jumper](https://www.amazon.co.uk/dp/B071Z31MQV?psc=1&ref=ppx_yo2ov_dt_b_product_details) device instead of bodging it myself.
<br> 

So now I have an external PSU providing SATA power to each Mellanox card in each machine. You can see I 3D printed a PCIe enclosure for these machines
(more on that in a later blog). 

![Scuffed power setup](/images/lenovo-mellanox-upgrade/scuffed-power-setup.jpg)

## Testing performance

### In Ethernet mode

Both machines `lilsleepy` and `bigwakey` are running Ubuntu 20.04 for testing, I already configured both Mellanox Cards to be in Ethernet mode instead of Infiniband. I changed the 
`/etc/netplan/00-installer-config.yaml` (why change the file for simple tests?) to:
```
# This is the network config written by 'subquity'
network:
  ethernets:
    # This is the onboard 1g NIC
    enp0s31f6:
      dhcp4: true
    # The ethernet device when Mellanox card is in Ethernet mode
    enp1s0:
      dhcp4: false
      addresses: [192.168.77.10/24]
```
I've given `192.168.77.10` for `lilsleepy` (the smaller M910q machine) and `192.168.77.20` for `bigwakey` (the more powerful M80q machine).

I didn't have to install any specific Mellanox driver packages for this to work on a fully updated Ubuntu 22.04. Remember I already preconfigured these cards 
for Ethernet mode, I'll go over the instructions for changing them between Infiniband and Ethernet modes later. 

#### `iperf3` TCP testing

Lets test network performance with `iperf3`, on `bigwakey` ([Using this for reference](https://fasterdata.es.net/performance-testing/network-troubleshooting-tools/iperf/)).
```
test@bigwakey:~$ iperf3 -s
```
```
test@lilsleepy:~$ iperf3 -c 192.168.77.20 -i 1 -t 30 -O -P 4
```
The arguments explained are:
 - `-i 1` - give test results every second
 - `-t 30` - test for 30 seconds
 - `-O` - Omits the "slow start" of a TCP test
 - `-P 4` - Runs the test over 4 parallel streams
<br>
This gave me _18.6Gb/s_ pretty consistently. Adding the reverse flag `-R` to test the results with transmission in the reverse direction, I was able 
to get _20.4Gb/s_ consistently, which makes sense with `bigwakey` being a more powerful machine. 

This is with a regular MTU of 1500, to maximise the performance I then enabled jumbo frames on both machines:
```
ip link set enp0s1 mtu 9000
```
With the same settings this gave a max bandwidth of _20.2Gb/s_ from `lilsleepy` to `bigwakey`, and _21.5Gb/s_ in reverse. 

#### `nuttcp` UDP Testing

I'm going to test UDP performance with nuttcp, as recommended [here](https://fasterdata.es.net/performance-testing/network-troubleshooting-tools/throughput-tool-comparision/). 
I must confess I'm no expert on UDP benchmarking, and I'm puzzled by the results. I have been following the recommendations for nuttcp testing on 10Gb link [here](https://fasterdata.es.net/performance-testing/network-troubleshooting-tools/nuttcp/).
<br>
For this testing I set both machines mtu to 9000. 

```
test@bigwakey:~$ nuttcp -S -u -l8948 -w4m  
```
```
test@lilsleepy:~$ nuttcp -u -l8948 -T30 -i1 -w4m 192.168.77.20
```
The arguments explained are: 
 - `-u` - Run in UDP mode
 - `-l8948` - Use a buffer length/packet size or 8948 (the referenced guide stated 8972 but I would get warnings from nuttcp for going higher than 8948)
 - `-w4m` - Use a window size of 4MB
 - `-T30` - Test for 30 seconds
 - `-i1` - Report results every second

The most performance I was able to get out of this was _9.1Gb/s_, without the larger mtu, buffer and window lengths specified the most I can get is _0.91Mb/s_. 
<br>

#### Testing disk to disk performance

Whilst we have a good idea of memory to memory network performance with iperf, a lot of interactions between these Lenovo machines in a cluster will be disk to disk. 
To get an idea of disk to disk performance (and encryption), I've borrowed a `dd` and `ssh` based response from [this SO question](https://askubuntu.com/questions/7976/how-do-you-test-the-network-speed-between-two-boxes).
```
ssh test@192.168.77.20 'dd if/dev/zero bs=1GB count=20 2>/dev/null' | dd of=/dev/null status=progress
```
With this I was getting 1.17Gb/s in throughput. It turns out this is a terrible method for testing performance!
Instead I decided to just make a large file and try to SCP the file between the two devices. 
```
fallocate -l 50G bigfile.data
scp ./bigfile.data test@192.168.77.20:~/
```
With this I was getting speeds of around 3.2Gb/s (around 400MB/s) which roughly corresponds to the read/write speeds of a SATA SSD. 
<br> 

This might seem annoyingly low, we're putting in a 40Gb/s link between these devices and we're still limited to SSD speeds. Why not just put an NVME SSD back
in and get 2-4Gb/s write/read speeds? Well because we would still be limited by the 1Gb/s onboard NIC, a 10Gb or 40Gb network card allows remote file read/writes
of 3-4 times faster. If we still want NVME speeds, we can have the devices use network storage from a high speed pool in our NAS, which would also offer other
neat features, like deduplication and backups. I still count this as a win!


### In Infiniband mode

I'm new to Infiniband so this is an adventure for me, but I'm excited to see what sort of memory to memory performance is possible with RDMA. 

#### Infiniband setup

Note: It is highly recommended you setup and use [mstflint](https://network.nvidia.com/support/firmware/update-instructions/) to upgrade your card to the latest firmware! 

To set up Infiniband on these machines, we first need to change the port mode on the cards to Infiniband mode. This requires downloading [Mellanox Firmware Tools (MFT)](https://network.nvidia.com/products/adapter-software/firmware-tools/).
Remember to check the checksum!
```
wget https://www.mellanox.com/downloads/MFT/mft-4.24.0-72-x86_64-deb.tgz
tar -xvzf mft-4.24.0-72-x86_64-deb.tgz
cd mft-4.24.0-72-x86_64-deb
sudo install.sh
```
Now we can initialize mst and inspect the device:
```
sudo mst start
lspci | grep Mellanox 
sudo mstconfig -d <insert PCIe lane> query
```
The LINK_TYPE_P1 and LINK_TYPE_P2 are currently set to 2 (Ethernet mode), we need to set them to 1 (Infiniband mode).
```
sudo mstconfig -d <insert PCIe lane> set LINK_TYPE_P1=1 LINK_TYPE_P2=1
sudo reboot now
```
Now we need to install and configure modules and packages for using RDMA ([reference](https://network.nvidia.com/pdf/prod_software/Ubuntu_20_04_Inbox_Driver_User_Manual.pdf)).
```
# Note these are the modules required for up ConnectX-3 cards, the modules for ConnectX-4 and beyond are different!
modprobe mlx4_en
modprobe mlx4_core
modprobe mlx4_ib

sudo apt install rdma-core libibmad5 ibutils ibverbs-utils
```

After this you can edit `/etc/rdma/modules/rdma.conf` and `/etc/rdma/modules/infiniband.conf` to pull in kernel modules for 
working with rdma. You will need to reboot after changing these. Make sure you have `ib_ipoib` enabled!

For Infiniband to work, one machine on the network must be the Subnet Manager. I've made `lilsleepy` the subnet manager here. 
To do this, install opensm and confirm the service is running:
```
sudo apt install opensm
sudo systemctl status opensm
```
You can now get interesting information about your infiniband link with tools like `ibstat`, `ibhosts`, `ibping` and `ibdiagnet`. 

Most software tools are not Infiniband aware, to ensure they still work we need to run IP over Infiniband (IPoIB).
If the `ib_ipoib` module was loaded, running `ip link` should show infiniband network devices with names beginning with `ibp`. These devices can 
be given IP addresses in `/etc/netplan/*.yaml` files. I've chosen to run a near identical config as before. 
```
# This is the network config written by 'subquity'
network:
  ethernets:
    # This is the onboard 1g NIC
    enp0s31f6:
      dhcp4: true
    # The ethernet device when Mellanox card is in Ethernet mode
    ibp1s0:
      dhcp4: false
      addresses: [192.168.77.10/24]
```

#### Benchmarking with `qperf`

One thing I've found, is that when testing RDMA with `qperf` you need IPoIB to have an address to point qperf at. I couldn't
find a non-IPoIB method to do it, though I'm sure there is a way. 

First we need to install `qperf` on both machines:
```
sudo apt install qperf
```
Then as with `iperf3` we need one machine (in this example `bigwakey`) to act as the server instance.
```
test@bigwakey:~$ qperf
```
Then run tests from a client instance:
```
test@lilsleepy:~$ qperf 192.168.77.20 -t 10 tcp_bw tcp_lat
```

There are a variety of different tests you can perform with `qperf` [listed here](https://linux.die.net/man/1/qperf). 

Here are the speeds I managed to achieve for each test:

| Test name | Explanation | Bandwidth/Latency |
|-------|---------------|------------|
| rds_bw | RDS streaming one way bandwidth | 17.2 Gb/s | 
| rds_lat | RDS one way latency | 12.4 us | 
| sctp_bw | SCTP streaming one way bandwidth | 3.5 Gb/s |
| sctp_lat | SCTP one way latency | 16.9 us |
| tcp_bw | TCP streaming one way bandwidth | 20.9 Gb/s | 
| tcp_lat | TCP one way latency | 13.5 us |
| udp_bw | UDP streaming one way bandwidth | 7.02 Gb/s |
| udp_lat | UDP one way latency | 13.4 us |
| rc_bi_bw | RC streaming two way bandwidth | 46 Gb/s |
| rc_bw | RC streaming one way bandwidth | 25.3 Gb/s |
| rc_lat | RC one way latency | 9.28 us |
| uc_bi_bw | UC streaming two way bandwidth | 46 Gb/s |
| uc_bw | UC streaming one way bandwidth | 25.2 Gb/s |
| uc_lat | US one way latency | 7.15 us |
| ud_bi_bw | UD streaming two way bandwidth | 38.8 Gb/s |
| ud_bw | UD streaming one way bandwidth | 23.1 Gb/s |
| ud_lat | UD one way latency | 7.22 us |
| rc_rdma_read_bw | RC RDMA read streaming one way bandwidth | 23.5 Gb/s |
| rc_rdma_read_lat | RC RDMA read one way latency | 7.91 us |
| rc_rdma_write_bw | RC RDMA write streaming one way bandwidth | 25.3 Gb/s |
| rc_rdma_write_lat | RC RDMA write one way latency | 7.39 us |
| rc_rdma_write_poll_lat | RC RDMA write one way polling latency | 1.52 us |
| uc_rdma_write_bw | UC RDMA write one way bandwidth | 25.2 Gb/s |
| uc_rdma_write_lat | UC RDMA write one way latency | 5.97 us |
| uc_rdma_write_poll_lat | UC RDMA write one way polling latency | 1.51 us |
| rc_compare_swap_mr | RC compare and swap messaging rate | 1.19 M/sec |
| rc_fetch_add_mr | RC fetch and add messaging rate | 1.18 M/sec |
| ver_rc_compare_swap | Verify RC compare and swap | 1.19 M/sec | 
| ver_rc_fetch_add | Verify RC fetch and add | 1.19 M/sec | 

Don't mistake me for someone who knows what most of these tests mean, I put them here for more knowledgable folks!
I understand that UD, UC and RC all different RDMA [Transport Modes](https://docs.nvidia.com/networking/display/RDMAAwareProgrammingv17/Transport+Modes). RDS is short for 
[Reliable Datagram Sockets](https://www.ibm.com/docs/en/aix/7.2?topic=protocols-reliable-datagram-sockets-over-infiniband-roce)
(over Infiniband). I am also unfamiliar with [Stream Control Transmission Protocol (SCTP)](https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol).
I will have to do some further reading!

The important sections for me are the TCP/UDP performance, and the various one way streaming bandwidth for RC, UC and UD transport modes.

From what I've read on the internet, most people find the IPoIB performs worse than regular Ethernet. In these tests 
there's only a mild difference of about 1 Gb/s difference in speeds. It's not quite a fair comparison because with 
`iperf3/nuttcp` considering I've tinkered and tweaked things like MTU. I'm sure similar tweaks can be done with IPoIB, but I will 
have to learn about that a later time. 

As far as Infiniband RDMA performance goes, it looks like bypassing the TCP/IP stack gives a boost of around 4-5 Gb/s.
This is cool, but it's the difference between 2.7GBs a second and 3.1GBs a second. The difference is not enough to make me
want to drop good old fashioned ethernet. 

## Conclusion

Whilst I knew the maximum data bandwidth through the PCIe Gen3 x4 lane would be 36Gb/s, there is a part of me that is a little
disapointed with the results. Here I am using a 40Gb link and only realistically getting 20Gb out of it for remote shares.
I thought perhaps I could "downgrade" to 25Gb cards, but they are much more expensive. Presumably because 40Gb is a bit of 
a dead end hardware-wise, 25/100Gb is the next generation (hopefully they will also be dropping to similar prices soon).  

I run TrueNAS Scale for my NAS at home and from what I've read there isn't much support for Infiniband. It can apparently
be hacked together in the shell, but it's not recommended and you can't do remote shares like NFS over Infiniband. 
I think in the near future I will stick with regular Ethernet on these cards. The nice thing is that whilst the maximum
bandwidth will fall in line with these tests, most of my 40Gb Mellanox cards are dual port. So I could run my "production"
networking over regular Ethernet, but then experiment with Infiniband connectivity separately, which I think would be quite
neat. 

So what have we achieved? What gain did we get from this upgrade?  
 - Remote disk to disk transfers between Lenovo Thinkcenters machines at SATA SSD speeds (~400MB/s)
 - In theory disk to disk between Lenovo Thinkcenters and a NAS NVME pool should run at 20-21Gb/s (slightly slower than Gen 3 NVME)
 - Memory to memory transfers between Lenovo Thinkcenter machines up to 20-21Gb/s 

I need to do further testing between my Lenovo machines and my NAS but all in all I'm fairly happy with the results
and I'm looking forward to having a much faster low power computing cluster. 

Until next time,<br>
Phil Stine


