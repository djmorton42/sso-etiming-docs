---
title: "SSO-FinishLynx Photofinish Setup Documentation"
description: "Setup, operation, and troubleshooting guides for our photofinish and lap timing equipment."
---

# Photofinish Setup Documentation

Practical guides for setting up, maintaining, and troubleshooting our photofinish and lap timing equipment. Each guide includes the relevant equipment, network settings, and verification steps.

## Setup diagrams

- **[SSO FinishLynx E-Timing setup]({{ '/etiming-setup-diagram.html' | relative_url }})**\
  View network, timing equipment, peripheral, and video diagrams, including equipment addresses and setup details to confirm.

## Setup guides

- **[E-Timing Router and LAN setup]({{ '/etiming-router-setup.html' | relative_url }})**  
  Configure the two ASUS routers so the timing desk and results office share one LAN over a 2.4 GHz bridge. Includes the address plan, DD-WRT settings, and verification.

- **[MYLAPS clock synchronization through Capture]({{ '/mylaps-capture-ntp.html' | relative_url }})**  
  Configure local DNS and NTP on Capture so the MYLAPS decoder can synchronize its clock on the isolated timing LAN. Includes setup instructions, meet-day checks, and troubleshooting.

- **[Starter timing cable: XLR to MYLAPS D-sub 15]({{ '/mylaps-xlr-db15-wiring.html' | relative_url }})**  
  Wire the starter pistol's three-pin XLR timing branch to the MYLAPS decoder's D-sub 15 auxiliary port. Includes the pin connections, the local 8-to-13 jumper, and the checks that confirm a start.

## Venue notes

- **[Venue notes]({{ '/venue-notes.html' | relative_url }})**  
  What to expect at each rink: network, camera placement, timing desk, cable runs, transponder loop, and overnight access.

<!-- Add future guides using the same pattern:
- **[Guide title]({{ '/guide-filename.html' | relative_url }})**  
  Briefly describe what the guide helps the reader do.
-->

## Using these guides

- Start with the guide's network and equipment details; confirm they match the setup in use.
- Complete its verification steps after changing settings.
- Make clock and synchronization changes outside an active race.
- Update the documentation when equipment, addresses, or procedures change.
