# alert_master_processor
This is how I do my alerts in Home Assistant with an LCARS twist.

Here's an ASCII flowchart showing the general flow:

```
┌──────────────────────┐
│ alert_initiator_*    │  ← User/system script (e.g. fire, AQI)
└────────┬─────────────┘
         │
         ▼
┌────────────────────────────────────────────┐
│ Set alert state helpers:                  │
│   input_select.alert_state_color          │
│   input_text.alert_state_reason           │
└────────┬───────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────┐
│ Alert Master Processor (AMP) Automation    │
│ Trigger: alert_state_color or reason change│
└────────┬───────────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Create scene.snapshot        │
│   (scene.pre_alert_lighting) │
└────────┬─────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│       Alert State Effects          │
└────────────┬────────────┬─────────┘
             │            │
             ▼            ▼
   ┌────────────────┐  ┌────────────────┐
   │ Lighting Effect │  │ Screen Effect  │
   │   via:          │  │   via:         │
   │ script.alert_   │  │ script.alert_  │
   │   effect_light  │  │   effect_screen│
   └────────────────┘  └────────────────┘
             │
             ▼
   ┌────────────────────────────────────┐
   │ Audio Alert (inline in AMP)        │
   │  - Based on input_text.alert_state_reason │
   │  - Plays appropriate audio clip    │
   └────────────────────────────────────┘

   [ Meanwhile… ]

┌────────────────────────────┐
│ alert_resolver_* scripts   │  ← User/system-triggered
└────────┬───────────────────┘
         ▼
┌──────────────────────────────────────────────┐
│ Reset alert state:                           │
│   input_select.alert_state_color = "none"    │
│   input_text.alert_state_reason = "none"     │
└────────┬─────────────────────────────────────┘
         ▼
┌──────────────────────────────────────────────┐
│ AMP detects reset state                      │
│ → Runs script.alert_effect_light_color_resolver│
│   - Turn off light effect scripts            │
│   - Wait until confirmed off or timeout      │
│   - Restore scene.pre_alert_lighting_state   │
│   - Log failure if unresolved after timeout  │
└──────────────────────────────────────────────┘

```

alert_state_color is an "input select" helper with four possible states, none, red, yellow, and blue. alert_state_reason has every possible reason for every possible alert:

![Screenshot 2025-06-27 174528](https://github.com/user-attachments/assets/86f1fa59-0569-4050-920b-bd2a7447c64d)

The Alert Master Processor monitors changes to these entities and acts accordingly. It's a complex automation.

```
alias: Alert Master Processor
description: >-
  Any entity, script, or automation wanting to set an alert state must do so
  through the AMP, by setting alert_state_color and alert_state_reason. The AMP
  detects changes to alert_state_color, which sets screen and lighting colors.
  alert_state_ reason determines the audio alert and triggers the specific
  script for that reason. When both states are set to none, the automation
  cancels lighting scripts and screen colors, and stops audio playback.
triggers:
  - alias: Monitor alert_state_color and alert_state_reason
    entity_id:
      - input_select.alert_state_color
      - input_select.alert_state_reason
    trigger: state
    from: null
    to: null
conditions: []
actions:
  - alias: If none, resolve all and stop
    choose:
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: none
          - condition: state
            entity_id: input_select.alert_state_reason
            state: none
        sequence:
          - alias: Turn off all _signal scripts; need to do this first
            action: script.turn_off
            metadata: {}
            data: {}
            target:
              entity_id:
                - script.alertstate_blue_aqi_hazardous_signal
                - script.alertstate_blue_aqi_very_unhealthy_signal
                - script.alertstate_red_tornado_warning_signal
                - script.alertstate_red_severe_thunderstorm_warning_signal
                - script.alertstate_red_severe_thunderstorm_warning_with_high_winds_signal
                - script.alertstate_yellow_severe_thunderstorm_watch_signal
                - script.alertstate_yellow_tornado_watch_signal
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id:
                - script.alert_effect_light_color_resolver
                - script.alert_effect_screen_color_resolver
            alias: Turn on resolver scripts; do not wait for them to finish
          - action: media_player.media_stop
            metadata: {}
            data: {}
            target:
              entity_id:
                - media_player.vlc_telnet
                - media_player.vlc_telnet_2
            alias: Stop VLC media players
          - action: logbook.log
            metadata: {}
            data:
              name: Alert Master Processor
              message: >-
                "none" state of alert_state_color and alert_state_reason was
                detected, stopping.
          - stop: if "none", then we should not activate anything else.
  - alias: >-
      Stores the state of the alert lights for retrieval when the state is
      resolved
    data:
      scene_id: pre_alert_lighting_state
      snapshot_entities:
        - light.kitchen_lightstrip
        - light.stairs
        - light.hallway
        - light.office_wall_left
        - light.office_wall_right
    action: scene.create
  - delay:
      hours: 0
      minutes: 0
      seconds: 2
      milliseconds: 0
  - alias: Choose alert lighting effect
    choose:
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: red
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_light_color_red
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: yellow
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_light_color_yellow
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: blue
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_light_color_blue
  - alias: Choose alert screen color
    choose:
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: red
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_screen_color_red
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: yellow
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_screen_color_yellow
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_color
            state: blue
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alert_effect_screen_color_blue
  - action: media_player.media_stop
    metadata: {}
    data: {}
    target:
      entity_id:
        - media_player.vlc_telnet_2
        - media_player.squeezeplay_2c_7b_a0_43_f4_e5
  - action: input_boolean.turn_on
    metadata: {}
    data: {}
    target:
      entity_id: input_boolean.alert_audio_active
    alias: Enable audio output until the user disables it
  - alias: Choose audio alert
    choose:
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: red_fire
        sequence:
          - action: script.alert_audio_reason_fire
        alias: fire
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: red_tornado_warning
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_red_tornado_warning_signal
        alias: tornado warning
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: yellow_tornado_watch
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_yellow_tornado_watch_signal
        alias: tornado watch
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: yellow_severe_thunderstorm_watch
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_yellow_severe_thunderstorm_watch_signal
        alias: severe thunderstorm watch
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: red_severe_thunderstorm_warning
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_red_severe_thunderstorm_warning_signal
        alias: severe thunderstorm warning
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: red_severe_thunderstorm_warning_with_high_winds
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: >-
                script.alertstate_red_severe_thunderstorm_warning_with_high_winds_signal
        alias: severe thunderstorm warning with high winds
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: blue_co
        sequence:
          - action: script.alert_audio_reason_co_detected
        alias: carbon monoxide
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: blue_airquality_hazardous
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_blue_aqi_hazardous_signal
        alias: air quality hazardous
      - conditions:
          - condition: state
            entity_id: input_select.alert_state_reason
            state: blue_airquality_very_unhealthy
        sequence:
          - action: script.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: script.alertstate_blue_aqi_very_unhealthy_signal
        alias: air quality very unhealthy
mode: restart
```
