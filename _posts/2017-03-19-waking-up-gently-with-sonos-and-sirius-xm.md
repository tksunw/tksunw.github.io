---
title: "Waking Up Gently with Sonos and Sirius XM and a Fade-In Alarm"
date: 2017-03-19 12:00:00 -0500
categories: [Automation]
tags: [sonos, siriusxm, python, iot, home-automation]
---

My wife and I use LIFX smart lights in our bedroom to simulate sunrise, with lights activating at sunrise and gradually brightening over 30 minutes for a gentler wake-up experience.

She requested extending this concept to our Sonos speaker system. The desired features included:

- Selection of any Sirius XM, Pandora, or Calm Radio station accessible via Sonos
- Customizable maximum volume setting
- Adjustable fade-in duration (ideally 30 minutes)
- iOS app control on iPhone or iPad

As extra credit, I wanted reverse functionality for a gradual volume decrease as a sleep timer.

## Initial Investigation

The native Sonos app alarms proved inadequate, offering only a 15-second fade-in rather than the requested 30-minute ramp-up. I discovered this was a recurring feature request in Sonos community forums, with unanswered requests spanning multiple years.

## Solution Development

I identified [SoCo](https://github.com/SoCo/SoCo), a Python library for Sonos control, which could accomplish approximately 60% of requirements via command-line automation through cron scheduling on a Linux server. This approach enabled:

- Scheduled program execution
- Volume adjustment across Sonos speakers or groups
- Station selection

## Limitations

iOS app orchestration remained unavailable through this method.

## Additional Features

- Holiday support via a `holidays.txt` file (YYYY-MM-DD format, one per line)
- Manual override capability via `/tmp/holiday` file creation for unplanned days off

## Code

The alarm script is published on GitHub at [https://github.com/tksunw/IoT/tree/master/SONOS](https://github.com/tksunw/IoT/tree/master/SONOS).

I remain open to discovering an iOS app solution that could fully orchestrate this automation until Sonos or competing home automation platforms add native support.
