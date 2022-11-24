> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [snippets.martinwagner.co](https://snippets.martinwagner.co/2021-08-21/mitmproxy-qemu)

> Recently, I wanted to take a look at the network traffic of an Android App that I was using. I did th......

Written by Martin  
on August 21, 2021

Recently, I wanted to take a look at the network traffic of an Android App that I was using. I did this a while back when, it was relatively easy to configure an HTTP Proxy and import a custom CA cert to mitm a HTTPS connection but was aware that it got more difficulty in the recent years. This guide describes the setup I used to still achieve my goal.

QEMU and Android-x86
--------------------

Since I didn’t want to install the entire Android SDK and emulator, I used [Android-x86](https://www.android-x86.org/) and QEMU to run an Android VM. [This repo](https://github.com/rexim/qemu-android-x86-runner) contains some nice default parameters to use. To start the installation process I ran:

```
$ qemu-img create -f qcow2 "android.img" 4G # new vm image

$ qemu-system-x86_64 -enable-kvm -vga std \
                   -m 2048 -smp 2 -cpu host \
                   -soundhw ac97 \
                   -net nic,model=e1000 -net user \
                   -cdrom "android-x68.iso" \ 
                   -hda "android.img" \
                   -boot d \
                   -monitor stdio


```

Networking
----------

Since we need to intercept the traffic of the VM, I modified the network config to not use the default usermode networking, but a tap device instead.

```
$ ip tuntap add dev tap0 mode tap user $USER
$ ip addr add dev tap0 192.168.1.1/24 # 192.168.1.0/24 is the network I choose for my VM
$ ip link set up tap0
$ iptables -t nat -A POSTROUTING -o enp42s0 -j MASQUERADE # mode -o to use your hosts ethernet interface


```

I also configured my host system to act as a NAT gateway, so the VM can reach the net.

Proxy setup
-----------

Since I don’t run a dedicated DHCP server for the VM, it is necessary to set a static IP in the Wi-Fi settings of the VM. I choose `192.168.1.128` and set `192.168.1.1` as default gateway (choose whatever you set for your tap0 device).

To redirect HTTP and HTTPS traffic to [mitmproxy](https://mitmproxy.org/) we have to insert two additional rules in our firewall:

```
$ # redirect request to port 80 / 443 to 8080 on the host
$ iptables -t nat -A PREROUTING -s 192.168.1.128 -p tcp --dport 80 -j DNAT --to-destination 192.168.0.233:8080
$ iptables -t nat -A PREROUTING -s 192.168.1.128 -p tcp --dport 443 -j DNAT --to-destination 192.168.0.233:8080
$ # start mitmproxy with the webgui. Listens on :8080 by default
$ mitmweb --mode transparent --showhost


```

Replace `192.168.1.128` with the IP of your VM and `192.168.0.233` with the IP of your host.

TLS pinning and Frida
---------------------

Most apps ignore user installed CA certs on Android, and some even explicitly pin the cert they use. This makes it hard for us to configure the mitmproxy CA that allows us to intercept HTTPS traffic. Luckily, there is a practical workaround: [Frida](https://frida.re/) can be used to patch Apps at runtime to accept any cert. Without going too much into detail, this is how it works:

Install Frida using `pip install frida-tools` and grab the latest `frida-server-*-android-x86_64` from the [release page](https://github.com/frida/frida/releases/). Connect to your emulator using ADB and upload the Frida server binary:

```
$ # ensure ADB and ADB over network are enabled in the Android developer settings
$ adb connect 192.168.1.128:5555
$ adb root
$ chmod +x frida-server-*
$ adb push frida-server-* /data/local/tmp/frida-server
$ adb shell /data/local/tmp/frida-server & # run frida-server in the background


```

This is all the setup needed for Frida. Now you can use it to inspect and patch Apps running on your emulator. The easiest way to disable all cert validation checks for the currently shown app is to run

```
frida --codeshare sowdust/universal-android-ssl-pinning-bypass-2 -U -F --no-pause


```

which injects [this](https://codeshare.frida.re/@sowdust/universal-android-ssl-pinning-bypass-2/) simple script. Now you should be able to see all HTTP and HTTPS request in the mitmproxy web interface.