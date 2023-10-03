---
title: "Sonoff 433mhz RF Bridge"
Description: "Hacking the Sonoff RF Bridge v2 for ESPHome."

# Pop an image hero at the top of the page.
heroImage: "/images/banner_image_lounge.jpg"
# Hero height. Should take any CSS value, default px.
heroHeight: "200px"
# Slide the hero vertically in case your image isn't designed specifically for
# this purpose.
heroVert: "-250px"
# Apply a Blur effect to the hero 
heroBlur: 4
# Apply a Sepia tone effect to the hero
heroSepia: 60
---

{{< imagesc src="stairs-light.jpg" alt="Stairs Light" position="left" style="width: 300px; height:400px; float:left; margin-right: 50px; border-radius: 8px;">}}

## The Problem

### Learning & Sending RF Codes

I'd selected this cheap chinese trash Water Wave Lights Projector thing mostly because of the extremely small form-factor. It's wired to turn on with the Starfield in the main bar. Naturally, the save-state feature only half works when it powers back on. It remembers the color setting, but if the motor is set to the slowest speed, that speed doesn't resume.

Well, it came with this remote control that's obviously throwing RF, not IR signals.

A little research pointed me at the Sonoff RF Bridge. A nifty little device that can:

- Connect to Wifi with an ESP chip, and can therefore be flashed with Tasmota or ESPHome or other Open Source firmware.
- Receive *AND* send RF signals, including some kind of RF learning capability.

Most of the guides on the internet are about automating passive responses. e.g. "Trigger a Home Assistant event when motion is detected." I want it the other way: "Fire an RF code when an event happens."

The problem is that there's a new version of this RF bridge, and Sonoff made it much more difficult to work with random undocumented chinese junk and their RF remotes. Since it took me all damn day to figure this out, I'm writing it down here. Mostly in case I need to do it again, but also in case someone else stumbles across it.

I'm going to mostly use other peoples pictures for this, since my thing works and I don't want to take it apart again.

---

At some point in like 2021, Sonoff updated the RF Bridge for no good reason. If you have one of these, ignore all the Tasmota and ESPHome instructions. If the instructions involve "Portisch", they won't work. You'll know you have one of these new ones if it has a white case.

#### YOU CANNOT USE PORTISCH WITH THE SONOFF RF BRIDGE V2.2 ANY TUTORIAL THAT INVOLVES PORTISCH WON'T WORK WITH THIS HARDWARE.{ .ctr-content }

---

{{< imagesc src="sonoff-board.png" alt="Sonoff board" position="right" style="width: 300px; height:300px; float:right; margin-left: 50px; border-radius: 8px;">}}

{{< imagesc src="sonoff-back-cover.png" alt="Sonoff Back Cover" position="right" style="width: 300px; height:300px; float:right; margin-left: 50px; border-radius: 8px;">}}

## Disassembly

Take the unit apart by removing the little rubber pads and the screws under those pads.

Once the cover is off, the board will pop right out. Might take a little wiggling, but there's no glue or other screws or anything.

Black/White case aside, it's DEFINITELY the v2 board, completely incompatible with most of the internet instructions that were all based on the v1 board. You can see that the antennas are embedded, and the electronics are pretty compact. The old version looked much more bolted-together. The change was probably mostly cost-savings, but there wasn't a great reason to break the hackability with this new chip.

Also, a dead giveaway is that it's labeled v2.2 in the top right corner.

---

## Flash ESPHome

### (or Tasmota, but you're on your own with getting that to work.)

I dicked around with Tasmota using the "sensors" build for hours, and it just wouldn't work. It sent RF codes, but didn't seem to capture incoming raw ones. You also need to create MQTT sensors in Home Assistant and it's just a pain to figure out. ESPHome is *much* easier.

#### 1. Create the ESPHome Device
  
First thing is to create the baseline ESPHome YAML file. Just enough to get it online. In your ESPHome dashboard, add a new device, and edit the YAML with the configuration below.

{{< details "ESPHome Configuration" >}}

```yaml
esphome:
  name: sonoff-rf
  platform: ESP8266
  board: esp8285

logger:

web_server:

api:
  encryption:
    key: !secret enc_key

ota:
  password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

ap:
  ssid: "Sonoff-Rf Fallback Hotspot"
  password: !secret ap_password

captive_portal:

status_led:
  pin:
    number: GPIO13
    inverted: yes

remote_receiver:
  pin:
    number: GPIO04
    inverted: True

  dump: all
  tolerance: 50%
  buffer_size: 2kb
  filter: 100us

remote_transmitter:
  pin: GPIO05
  carrier_duty_percent: 100%
```

{{< /details >}}

**Don't just copy/paste.**
You'll need to have the right entries in your esphome secrets.yaml, or make sure you replace the entries to the left.

#### 2. Flash the ESPHome binary to the Sonoff

  There's no OTA method to replace Sonoff. You MUST connect a serial programmer. If you're into this already, you should know how to do that.

  Click Install, and select manual download. Newer versions of ESPHome will ask if you want "modern" or "Legacy" format. I'm pretty sure it doesn't matter, but I used modern, no problems. ESPHome will generate a binary and you can use the ESPHome Web Flasher or whatever preferred method to flash it.

  The web interface is usually the easiest. Make sure you hold the reset button on the board when you power it up so it'll go into programming mode.

  Make sure it boots up, connects to your wifi, and ESPHome sees it online before continuing.

---

#### 3. hardware Hacks

  This is the key magic to getting the new version of the boards to work. Sonoff changed one of the chips, and this hack bypasses that.

  {{< imagesc src="sonoff-hacked-board1.png" alt="cut traces" position="left" style="width: 300px; height:296px; margin-right: 50px; float:left;">}}

1. Cut Traces

    There are a couple problems with the board design, preventing this from working. [Fortunately, people way smarter than me figured out how to solve it.](https://community.home-assistant.io/t/new-sonoff-rf-bridge-board-need-flashing-help/344326/18)

    Instead of cutting those teeeny little USB traces, you can cut them on the back-side of the board. Further down that page, you'll find the picture to the left. It's an alternative that's much easier to cut without doing more damage, plus a third trace that needs to be cut anyway.

    Use an exacto or a small screwdriver or something to make two small cuts on the bottom of the board. Make sure its deep enough to cut the trace, but not so deep it cuts any mid layers (if there are any, I dunno.)

    The lower two traces are USB data ports. Any data on those potentially interferes with the RF data.

    I have no idea what the upper trace is for.

    Note: I did not cut the line next to the RST in the pic below. Seems to work fine.

2. Solder the bypass wires in place

    {{< imagesc src="sonoff-hacked-board2.png" alt="cut traces" position="right" style="width: 300px; height:296px; margin-left: 50px; float:right;">}}

    Sonoff adds a chip between the transceiver and ESP to control Sonoff devices. On the older RF Bridges, there's a second firmware called 'Portisch' that got flashed to that chip to expand support outside the Sonoff product lines.

    The new version of the RFBridge uses a new chip that isn't compatible, and that's why we have to so this part of the hack.

    What I think this hack does is bypass that middle control chip, and connects the ESP directly to the RF transceiver.

    On the site linked above with this hack, one of the posts notes:

    In this mod, the existing encoder/decoder chip is not disconnected from either the transmitter nor the receiver. From what I could measure, this revision of the board has a 1k resistor between the encoder chip and the modulator, so the ESP “wins” anyway. It works perfectly fine for me, but do at your own risk.

    I'm not entirely sure what that means, but it seems important.

    **If it isn't clear in the picture:**

    - Solder a resistor that goes from USBRXD hole to the bottom left pin on the 8-pin chip.
    - Solder a resistor that goes from the USBTXD hole to the top left pin on the 6-pin chip. 
    - Both resistors should be 180-680 ohms.
    - Or just use wires. Works fine for me.
    - Double and triple check that the solder is not bridging anything. There's a few tiny capacitors around the 6-pin chip and it's easy to get a trace contacting those.

#### Capturing RF Codes

1. Dumping codes to ESPHome Logs

    There's no guarantee any of this will work. If your device uses rolling codes, it won't work. (That's true for the stock Sonoff too.) Most inexpensive 433Mhz stuff should be readable, but who knows?

    - Navigate to your ESPHome web, and open the logs for your device. You'll see any random RF traffic dumping through.
    - Press the button on your remote (or whatever sends the RF code) and watch for the unusual log entry.

    If you're not getting anything, then the device might not be using a known protocol. You might have better luck with raw output. Change the YAML above from "dump: all" to "dump: raw". "All" seems to be "all known protocols", raw is "everything we get that's a valid RF packet".

    {{< imagesc src="esphome-rf-logs1.png" alt="ESPHome Logs" position="center" >}}

2. Analyzing RF Codes

    If you have dump:all in the above YAML, then it'll tell you the protocol if it's recognized. No guarantees. You'll notice that the log says "Received Pronto:". That's the Pronto protocol.  You can edit the YAML to filter to just the one you want and quiet the logs. Wasn't helpful for me, since my house is apparently flooded with this Pronto radiation.

    Regardless, this is where it gets really fucking annoying. You're going to want to press that button several times. Like 20 or more. Do a few quick presses, and a few longer presses, and a few normal presses. Do some spaced out, and some quickly. Then download the logs, and start analyzing them. Sometimes the ESPHome logs will stop working and the device will reset. I'm guessing that's a memory problem. if that happens, save what you have, then close the logs and start over. 

    In my case, I noticed a few things: 

    - All the pronto junk started with "0000 006D" and ended with "06C3".
    - Most but not all the responses to the button press did the same.
    - The example listed on the ESPHome Remote Transmitter page for Pronto ALSO had the same.
    - Therefore, there's a good chance that valid codes follow that pattern.

    I also noticed that some of the button responses would not follow that pattern, but would do so across lines.

    There's a lot of inconsistency in the captured codes. Some are just incomplete, some are weirdly short, sometimes it seems to burst noise until I hit the button again.

    So I started copy/pasting and gluing lines together until I found a few reasonably consistent code strings. They're rarely identical, but apparently that's just part of code learning and normal.

    The main weird thing is that the length of the codes were inconsistent. Some lines would be 139 blocks long, and some would be as much as 145 blocks. I'm not sure where the extra data came from.

    Eventually through trial and error, I found one that worked, and the remaining part of the puzzle snapped into place. For each button on the remote, I was looking for the "0000 006D ... 06C3", with exactly 141 of those 4-character groups.

#### Sending RF Codes

To test the RF code you found, you'll want to copy it exactly like in your log.

Create a new block at the end of your YAML that looks like this:

{{< details "ESPHome RF Code" >}}

```yaml
button:
  - platform: template
    name: Water Light Speed Up

    on_press:
      - remote_transmitter.transmit_pronto:
          data: "0000 006D 0043 0000 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 0017 0018 0017 0018 0017 0018 003E 0018 003E 0018 0017 0018 0017 0018 0017 0018 003E 0018 003E 0018 003E 0018 0017 0018 0017 0018 003E 0018 003E 0018 003E 0018 0016 003D 015B 00AA 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 0017 0018 003D 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 003E 0018 0017 0018 0017 0018 0017 0018 003D 0018 003D 0018 0017 0018 0017 0018 0017 0018 003D 0018 003D 0018 003D 0018 0017 0018 0017 0018 003D 0018 003D 0018 003E 0018 0015 06C3"
```

{{< /details >}}

The data field is all one line.

You'll want to reference the Remote Transmitter Actions site to get the correct action for on_press and the correct format for data.

You can test the code by navigating to the IP address your router assigned the device, or you can add it to Home Assistant, and click the button that appears there.

#### Home Assistant

Home Assistant should auto-discover the device. If not, just go to Settings -> Devices and add a new integration. You'll need to supply the IP address and the encryption key you supplied to the YAML.

That's basically it. Save and install the YAML. If you captured the correct code, then your RF signal should blast forth from the Sonoff device when you press the button in Home Assistant.
