---
title: "Automating Holidays"
Description: "Get HomeAssistant to sense the holiday seasons."

# Pop an image hero at the top of the page.
heroImage: "/images/banner_image_lounge_crop.jpg"
# Apply a Blur effect to the hero 
heroBlur: 4
# Apply a Sepia tone effect to the hero
heroSepia: 60
---

## The Problem

Home Assistant can tell you if it's Thanksgiving if you have a calendar plugin configured with Thanksgiving scheduled. But the included Calendar plugin doesn't have that pre-scheduled, and what if you don't want to use a Cloud calendar service? The date sensor isn't very good with date math, and getting "Thanksgiving" to a date is difficult because it's not a fixed day.

## The Solution

### Template Sensors and Date Math

I found most of this on the Home Assistant Forums, but I've lost the link.

The YAML below will set a "holiday-season" sensor's state to "christmas" between the day after Thanksgiving and Jan 5th.

It'll set the state to "halloween" in October.

Otherwise it sets the state to "none". Simple enough.

{{< details "Home Assistant Automation Configuration" >}}
```yaml
- trigger:
  - platform: time_pattern
    # This will update every night
    hours: '0'
    minutes: '0'
  - platform: event
    event_type: event_template_reloaded
  - platform: homeassistant
    event: start

  sensor:
    - name: "holiday-season"
      state: >
        {%- set today = now().date() -%}
        {%- set month, week, day = 11, 4, 3 -%}
        {%- set temp = today.replace(month=month, day=1) -%}
        {%- set adjust = (day - temp.weekday()) % 7 -%}
        {%- set temp = temp + timedelta(days=adjust) -%}
        {%- set thanksgiving = temp + timedelta(weeks = week - 1) -%}
        {%- set jan6th = temp.replace(month=1, day=6) -%}

        {% if today <= jan6th or today > thanksgiving -%}
          {%- set season = "christmas" -%}
          christmas
        {%- elif today.month == 10 -%}
          {%- set season = "halloween" -%}
          halloween
        {%- else -%}
          {%- set season = "none" -%}
          none
        {%- endif -%}
```

{{< /details >}}

---

### Example Use Cases

#### Dashboards

Using the [Card Mod](https://community.home-assistant.io/t/card-mod-add-css-styles-to-any-lovelace-card/120744) plugin, you can style dashboard cards and use this sensor to change your themes.

{{< imagesc src="ha-holiday-dashboard.png" alt="Example Dashboard" style="width: 300px; height:238px; border-radius: 8px;">}}

**How to do this in a dashboard:**

#### Selecting a Scene

Using scenes whenever possible, rather than trying to capture everything manually is the best way to handle lighting and switches and such. In this case, rather than activating a scene, trigger an automation that activates the appropriate scene.

{{< imagesc src="ha-dash-config.png" alt="Example Dashboard" style="width: 500px; height:241px; margin-right: 50px; border-radius: 8px;">}}

This also works for playlists, videos, whatever.

{{< details "The YAML version of the above configuration" >}}
```yaml
alias: Select Holiday Scene
sequence:
  - choose:
      - conditions:
          - condition: state
            entity_id: sensor.holiday_season
            state: christmas

        sequence:
          - service: scene.turn_on
            target:
              entity_id: scene.bsow_bar_christmas_scene
            metadata: {}

      - conditions:
          - condition: state
            entity_id: sensor.holiday_season
            state: halloween

        sequence:
          - service: scene.turn_on
            target:
              entity_id: scene.bsow_bar_halloween_scene
            metadata: {}
    default:
      - service: scene.turn_on
        target:
          entity_id: scene.bsow_bar_default_scene
        metadata: {}

mode: single
icon: mdi:palette-swatch-variant
```

{{< /details >}}
