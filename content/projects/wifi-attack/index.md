---
title: "Wifi Deauthentication Attack Detection/Localization"
date: 2025-11-10
tags: ["Wifi", "ESP32", "Raspberry Pi"]
github: "https://github.com/scast3/deauth_detect"
tools: ["Wireshark"]
status: "complete"          # complete | in-progress
weight: 1                   # lower = appears first in listings
---


## Overview

Developed an intrusion detection system to detect and triangulate the source of WiFi jamming via deauthentication attacks, common in network disruptions. ESP32(s) scan for deauth/dissoc packets in promiscuous mode, relaying alerts to a Raspberry Pi server for logging and analysis. Use Flipper Zero (with WiFi module/ESP32 marauder) to simulate deauthentication attacks in a lab environment. RSSI localization done via least squares multilateration.

<div style="width: 500px; margin: 0 auto; text-align: center;">

![diagram](diagram.jpg)

</div>


[View on GitHub →](https://github.com/scast3/deauth_detect)