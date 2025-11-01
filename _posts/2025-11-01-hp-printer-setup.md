---
layout: post
title: "Setup (again): HP Laser MFP 136nw "
tags: [personal]
comments: false
---

A few weeks back, I had to reset my router for firmware updates. 

And because of that, some devices on my local network, in this case my HP printer stopped working on Wi-Fi.

I followed the following steps today, to make it work again.  
([MJB](https://janusworx.com) helped lots! Thank you so much! Very grateful!)

- I realised the IP (http://192.168.1.z) that was assigned to my HP Printer (before router reset) is now taken by some other device on my local network because of DHCP dynamically assigning IPs.

- I connect to the HP Printer via Ethernet, access the HP Printer configuration page on the last assigned IP (http://192.168.1.z).
  (Because now I am connected to the HP Printer via Ethernet, the router give preference to the HP Printer on the above IP, even though DHCP Server had assigned this IP to another device on my network.)
  
- I login to my HP Printer configuration page.  
  I go to "Settings > Network Settings > Wi-Fi".  
  On this menu page, click on "Wi-Fi Settings > Advanced Settings > Network Setup".  
  Go to "SSID" and hit "Search List" and "Refresh".  
  From the drop down, pick the Wi-Fi SSID I want to connect to, at the bottom, pick "Network Key Setup" and put the updated SSID password in there (both in "Network Key" and "Confirm Network Key").  
  Don't forget to hit "Apply".

- Now other thing I have to fix is that the IP address is still assigned by the Router's DHCP server to another device on LAN.  
  I need to assign a proper IP to my HP Printer outside the range of IPs available to DHCP server to assign to devices dynamically.
  
- For that, go to the Router admin page, login and go to "Local Network > LAN > IPv4".  
  Then go to the section "DHCP Server" and change "DHCP Start IP Address" and "DHCP End IP Address" respectively to some "192.168.1.a" and "192.168.1.b" and hit "Apply".  
  With this the router will now have IP "192.168.1.a-1" and DHCP server will only be able to assign dynamically IPs to devices within the assigned pool only.  
  (with this, what I am trying is to limit the pool of IPs available to DHCP server so that I can assign an IP ("192.168.1.b+1") to HP Printer outside the limits of this available DHCP server IP pool manually. So, that the printer IP doesn't conflict with any other device IP assigned by DHCP server.)
  
- Now login back to the Printer configuration page,  go to "Settings > Network Settings > TCP/IPv4".  
  Here, in the "General" section, pick "Manual" under the "Assign IPv4 Address" option.  
  And manually assign the following - (1) "IPv4 Address: 192.168.1.b+1", (2) "Subnet Mask: 255.255.255.0", and (3) "Gateway Address: 192.168.1.a-1" (should match the router IP address) to HP Printer.  
  And hit "Apply".  
  With this, the HP Printer configuration page itself will reload to the new assigned IP address url (http://192.168.1.b+1).
  
- After above steps, I then remove the Ethernet from the HP Printer, restart it.  
  And check if I am still able to access the HP Printer on the assigned IP via Wi-Fi (http://192.168.1.b+1).  
  Yes! It worked now!

- Then now I need to test, whether printing works on Wi-Fi.  
  I am on an OpenSUSE Tumbleweed machine.  
  I go to the "Settings > Printers" page.  
  I have to make sure that my printer is showing up there.  
  (It wasn't before, I had to once manually add a printer and pick up the latest available matching model from the available database, but that's not needed after my steps below.)

- Yes, printer shows up. I gave a test print job. Printing on Wi-Fi is working now.

- But Scanning still doesn't work. Neither on Wifi, nor on Ethernet.  
  My system just doesn't detect the scanner on my HP Printer at all.

- Now, I go back to HP Printer configuration page (http://192.168.1.b+1).  
  Go to "Settings > Network Settings" and ensure that "AirPrint" and "Bonjour(mDNS)" both are enabled.

- Now, I need to do a few things at the OS level.  
  Install (or ensure if already installed) the following set of packages.

  ```
  # install required packages 
  
  sudo zypper install avahi avahi-utils nss-mdns
  sudo zypper install hplip hplip-sane sane-backends
  sudo zypper install sane-airscan ipp-usb

  # enable, start and verify status of avahi daeomon (make sure I use `-l` flag to have all available information from the service status output)

  sudo systemctl enable avahi-daemon
  sudo systemctl start avahi-daemon
  sudo systemctl status -l avahi-daemon.service

  # make sure I have all avahi-related tools as well
  
  which avahi-browse
  which avahi-resolve
  which avahi-publish
  rpm -ql avahi | grep bin # gives me `/usr/sbin/avahi-daemon` and `/usr/sbin/avahi-dnsconfd`


  # ensure firewall is allowing mdns and ipp

  sudo firewall-cmd --permanent --add-service=mdns
  sudo firewall-cmd --permanent --add-service=ipp
  sudo firewall-cmd --reload
  sudo firewall-cmd --info-service=mdns
  sudo firewall-cmd --info-service=ipp

  # and restart firewall
  
  sudo systemctl restart firewalld
  sudo systemctl status -l firewalld

  # now check if avahi-browse can see devices advertised by my HP Printer

  avahi-browse -a | grep HP
  ## output something like following. I need to make sure it has `_scanner._tcp` and `_uscan._tcp` showing up
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _ipp._tcp            local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _scanner._tcp        local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _uscan._tcp          local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _uscans._tcp         local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _http._tcp           local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _pdl-datastream._tcp local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _printer._tcp        local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _http-alt._tcp       local
    + wlp0s20f3 IPv4 HP Laser MFP 136nw (xx:xx:xx)                 _privet._tcp         local

  # if device shows up, then check is scanner is responding on the network

  ping -c3 192.168.1.x

  curl http://192.168.1.x:8080/eSCL # any xml ouput is fine as long as there's something

  # final check
  
  scanimage -L
  ## it should list something like:
  device `airscan:xx:xx Laser MFP 136nw (xx:xx:xx)' is a eSCL HP Laser MFP 136nw (xx:xx:xx) ip=192.168.1.x
  ```

- At this point, the "Scan Documents" app should be detecting the scanner on my HP printer (it did!)

- Also, with Avahi working, my OS system "Settings > Printers" also got a HP Printer added automatically with the correct model name etc.  
  (Scanner also, although that doesn't show up as a menu item in the system settings.)
