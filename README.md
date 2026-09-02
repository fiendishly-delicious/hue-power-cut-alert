# Home Assistant Blueprint: Hue Lights - Get Alerted If Wall Switch Is Turned Off

This has bugged me for years. Guests — and family who should know better — flip the wall switch off on their way out of a room. The Hue bulbs lose main power, and from that moment the lights in that room are dead to automations: motion does nothing, schedules do nothing, voice commands do nothing. Home Assistant reports no error, because as far as it knows nothing happened. You find out hours later when you walk into a dark bathroom and wonder why the automation "broke". And when you do flip the switch back on, the bulbs come up in whatever state they were left in rather than the state your automations would have set them to be.

This blueprint creates an automation that promptly sends you a notification after someone flipped the wall switch, and optionally nags you until the switch goes back on.

---

## Get the blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-loaded.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffiendishly-delicious%2Fhue-power-cut-alert%2Fblob%2Fmain%2Fhue_bulb_power_cut_alert.yaml)

Or in Home Assistant go to **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste:

```
https://github.com/fiendishly-delicious/hue-power-cut-alert/blob/main/hue_bulb_power_cut_alert.yaml
```

---

## Why this is easy to miss

A wall switch cuts power — it never sends an "off" command. The bulb doesn't report anything, it just stops answering. There's no goodbye message, so nothing in Home Assistant flags an error. The room quietly stops responding to motion, schedules and voice, and you find out when you walk into the dark.

This blueprint watches one bulb on that circuit and tells you when it stops being reachable. It holds for a couple of minutes first, so a brief mesh hiccup or a hub restart doesn't wake your phone for nothing.

## Which entity to watch

Point it at the bulb's own light entity and it alerts when that entity goes `unavailable`. That works and needs no setup at all.

If you run Hue through the bridge, there's a sharper signal available. The Hue integration creates a **Zigbee Connectivity** diagnostic sensor for every bulb, reporting `connected`, `disconnected`, `connectivity_issue` or `unidirectional_incoming`. It speaks about one thing only: whether the bridge can still reach that bulb. `unavailable`, by contrast, is Home Assistant's all-purpose "I currently know nothing about this thing" — it also goes up when HA restarts, when the integration reloads, or when the bridge drops off the network. The connectivity sensor is the narrower and more dependable of the two, and the blueprint accepts either.

## Optional: enable the Zigbee Connectivity sensor

Hue creates this entity for every bulb but leaves it **disabled by default**, which is presumably why this problem is so persistent — the entity that speaks most clearly about it is hidden. To turn it on:

**Settings → Devices & Services → Hue → the bulb's device → the "+N entities not shown" link → Zigbee Connectivity → gear icon → Enabled → Update**

Give it about 30 seconds to appear, then pick it in the blueprint in place of the light.

Either way, the blueprint checks itself at runtime. If the entity it watches is later removed or renamed, then on every Home Assistant restart and every automation reload it messages you to say it is **not active** and how to fix it — rather than sitting there silently doing nothing, which is the failure mode I was trying to escape in the first place.

## Pick one bulb, not a room

Choose a single physical bulb connected to the switch. Any one bulb on the circuit is enough, since they all share it.

Don't pick a room, zone or group. Their entity stays "available" no matter what happens to the bulbs inside them, so it can never tell you about this.

For more than one switch, create one automation from the blueprint per switch. Each gets its own name, canary, recipients and reminder schedule, and there's no limit.

## Options

- **Bulb to monitor** — one bulb on the switched circuit: either the light entity itself, or that bulb's Zigbee Connectivity sensor if you use Hue and have enabled it.
- **Name of Light Switch** — optional. Used as the alert title, as *"<name> Is OFF!"*. Leave it blank and it uses the bulb's own name plus "Switch", e.g. *"Guest Bath 1 Switch"*. Type your own if you'd rather the notification to use something else, like "Downstairs Bathroom".
- **Canary Bulb** — optional, but highly recommended. A canary bulb is a bulb elsewhere in the house, on a different switch that isn't normally turned off. If the hub, your network, or the whole house's power drops, every bulb goes unreachable at once; so if the canary bulb is also not answering, the automation does not alert, because it's more likely a hub or power problem rather than a wall switch. Leave empty to skip the check.
- **Confirm for** — how long the bulb must stay unreachable before you're told. Default 2 minutes.
- **Who to notify** — one or more `notify` entities (the companion app on each phone). Add as many as you like.
- **Keep reminding me** — off gives one alert per switch-off; on nags until the switch goes back on. I'd recommend on, because the first alert usually lands when you can't act on it.
- **Remind me every** — the reminder interval.
- **Only remind me after** / **And stop reminding me at** — the reminder window, hourly between 08:00 and midnight by default. Outside the window you still get the *first* alert immediately — you just don't get nagged again until the window reopens. Set the end time equal to or earlier than the start to mean "until midnight".

The reminder loop ends the moment the bulb is reachable again — I measured it exiting 13 milliseconds after connectivity came back. There's deliberately no notification when the switch goes back on: you flipped it, you know.

## Feedback Welcome

There are two things I'd particularly like to hear about:

1. **Does your light entity go `unavailable` when you cut a switch?** Mine never do, despite the integration source suggesting it should. I'd like to know whether that's a quirk of my setup or common.
2. **Detection timing.** I only have one bridge's behaviour to go on, and I'd be interested to know whether the bridge's own self-discovery interval is consistent for other people.

---

**Like this?  Buy me a cup of coffee!**

[![Donate with PayPal](https://www.paypalobjects.com/en_US/i/btn/btn_donate_LG.gif)](https://www.paypal.com/donate/?business=F4BNEAREV25WC&no_recurring=1&item_name=Thanks+for+the+donation%21&currency_code=USD)

---

## Install it

Click to add the blueprint straight to your Home Assistant:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-loaded.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffiendishly-delicious%2Fhue-power-cut-alert%2Fblob%2Fmain%2Fhue_bulb_power_cut_alert.yaml)
