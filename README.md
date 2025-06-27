# alert_master_processor
This is how I do my alerts in Home Assistant with an LCARS twist.

Here's an ASCII flowchart showing the general flow:

┌──────────────────────┐
│ alert_initiator_*    │  ← Scripts or automations
│ (e.g. fire, tornado) │
└────────┬─────────────┘
         │
         ▼
┌────────────────────────────────────────────┐
│ Set:                                      │
│   input_select.alert_state_color          │
│   input_text.alert_state_reason           │
└────────┬───────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────┐
│ Alert Master Processor (AMP) Automation    │
│ Trigger: state change to alert_* helpers   │
└────────┬───────────────────────────────────┘
         │
         ▼
  ┌──────────────────────────────┐
  │ scene.create                 │
  │ (pre_alert_lighting_state)  │
  └────────┬────────────────────┘
           │
           ▼
     ┌──────────────┐
     │ Lighting     │──────┐
     │ Effects:     │      │
     │   script.alert_effect_light_color_* │
     └──────────────┘      │
                           ▼
                     ┌──────────────┐
                     │ Screen       │
                     │ Effects:     │
                     │   script.alert_effect_screen_color_* │
                     └──────────────┘
                           ▼
                     ┌──────────────┐
                     │ Audio Alerts │
                     │   script.alert_audio_reason_*       │
                     └──────────────┘

   [ Meanwhile… ]

┌────────────────────────────┐
│ alert_resolver_* scripts   │  ← User/manual/system triggered
└────────┬───────────────────┘
         ▼
┌────────────────────────────┐
│ Set alert state to "none": │
│   input_select.alert_state_color = none     │
│   input_text.alert_state_reason = ""        │
└────────┬───────────────────┘
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │ AMP automation detects reset to "none"       │
  └────────┬─────────────────────────────────────┘
           ▼
  ┌──────────────────────────────────────────────┐
  │ script.alert_effect_light_color_resolver     │
  │ - turn off alert light scripts               │
  │ - repeat loop to confirm shutdown            │
  │ - log failure if needed                      │
  │ - restore pre-alert lighting scene           │
  └──────────────────────────────────────────────┘
