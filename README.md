# Blueprint: know when someone turns a Hue bulb off at the wall switch

This has bugged me for over ten years of running Hue. Guests — and family who should know better — flip the wall switch off on their way out of a room. The bulbs lose mains power, and from that moment the room is dead: motion does nothing, schedules do nothing, voice commands do nothing. Home Assistant reports no error, because as far as it knows nothing happened. You find out days later when you walk into a dark bathroom and wonder why the automation "broke". And when someone does flip the switch back on, the bulbs come up in whatever state they were left in rather than the state your automations think they're in.

This blueprint tells you as soon as it happens, and nags you until the switch goes back on.

---

## Get the blueprint

The blueprint lives here: **[github.com/fiendishly-delicious/hue-power-cut-alert](https://github.com/fiendishly-delicious/hue-power-cut-alert)**

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-loaded.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffiendishly-delicious%2Fhue-power-cut-alert%2Fblob%2Fmain%2Fhue_bulb_power_cut_alert.yaml)

Or in Home Assistant go to **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste:

```
https://github.com/fiendishly-delicious/hue-power-cut-alert/blob/main/hue_bulb_power_cut_alert.yaml
```

---

## Why the obvious approaches don't work

I tried the obvious things first. None of them fire.

**Watching the light entity for `off`.** A wall switch cuts power — it never sends an "off" command. The bulb doesn't report anything, it just stops answering.

**Watching it for `unavailable`.** This is the one everybody tries, and it's the one that fooled me longest. I've now cut a switch with full recorder history running, and the light entity never reliably showed `unavailable` — not during the outage, not minutes later, sometimes not at all. 

Reading the Hue integration source, a bulb's availability is *supposed* to be calculated from exactly the zigbee connectivity information I'll describe below — which means on a correctly-behaving system the light entity *should* go unavailable, at the same moment that there is no zigbee. In practice, my hue bulbs do not. So I can only tell you what my system does. **If yours behaves differently I'd genuinely like to know** — see the bottom of the post.

There's a second reason to prefer monitoring for zigbee connectivity, even if your light entity does accurately track availability: `unavailable` is Home Assistant's all-purpose "I currently know nothing about this thing". It also goes up when HA restarts, when the Hue integration reloads, and when the bridge drops off the network. It's a vaguer signal for the same fact.

**Watching for *any* state change at all.** I had a state trigger with no `to:` or `from:` on that light, which fires on attribute changes too. Nothing. Checking `last_reported` — which bumps on any update whatsoever, attributes included — does not update when power is cut to the bulb.

The root cause is that **the bridge is never told when a bulb loses power**. There's no goodbye message. The bulb simply stops responding, and the bridge doesn't go looking until it has a reason to.

## What actually works

The Hue integration creates a **Zigbee Connectivity** diagnostic sensor for every bulb, reporting `connected`, `disconnected`, `connectivity_issue`, or `unidirectional_incoming`. That sensor tells the truth. However, it's also **disabled by default**, which is presumably why this problem is so persistent — the one entity that reliably detects it is hidden.

The blueprint creates an automation that watches that zigbee connectivity sensor, holds for a couple of minutes to ride out mesh hiccups, and then send you an alert notifying you that the switch for that light has been switched off.

## The part that took longest to work out: broadcast vs. direct

The bridge only discovers a dead bulb when it tries to talk to that bulb **and expects an answer**. It only expects an answer when it addresses that one bulb on its own — a direct message. When your lighting automation turns off a whole *room*, that goes out as a single broadcast to the group. **A broadcast needs no reply, so a missing reply tells the bridge nothing.**

I had assumed the opposite. I originally thought that the Hue bridge would send something back to the bulb after a group broadcast. It does not.  

So if your lighting automation turns off a Hue group, room, or zone — or even by scene — that turn-off will *not* reveal the dead bulb. You're left waiting on the bridge's own housekeeping, which is slow and, in my testing, not consistent enough to put a number on.

## What the blueprint does about it

It sends the direct message for you.

The moment the monitored bulb is told to turn off — by your own automation, a scene, the Hue app, a wall dimmer, however it happens — the blueprint waits two seconds and then sends **one more turn-off, addressed to that single bulb**.

You never see it, because the bulb was being turned off anyway. If the bulb is healthy, nothing changes. If the switch is off, the bridge gets no reply, marks it unreachable within seconds, and you're told as soon as your "Confirm for" time has elapsed.

That turns "maybe HA notices eventually" into "HA finds out moments after the room next turns itself off" — which is exactly when it starts to matter, because that's when the room was going to go dark anyway.

---

## ⚠️ You must enable the Zigbee Connectivity entity first

**The blueprint's automation will not work until you do this.** For each bulb you want to monitor:

**Settings → Devices & Services → Hue → the bulb's device → the "+N entities not shown" link → Zigbee Connectivity → gear icon → Enabled → Update**

Give it about 30 seconds to appear.

When you pick a light in the blueprint — you are telling the automation to monitor that light's zigbee connectivity sensor directly. **if the dropdown in the automation is empty, you haven't enabled zigbee connectivity for any bulbs yet** Hue rooms, zones and groups do not appear, because they have no such sensor — their entity stays "available" no matter what happens to the bulbs inside them.

The blueprint also checks at runtime. If the sensor is later disabled or removed, then on every Home Assistant restart and every automation reload it messages you to say it is **not active** and how to fix it — rather than sitting there silently doing nothing, which is the failure mode I was trying to escape in the first place.

## Pick one bulb, not a room

Choose a single physical bulb connected to the switch. Any one bulb on the circuit is enough, since they all share it.

For more than one switch, create one automation from the blueprint per switch. Each gets its own name, canary, recipients and reminder schedule, and there's no limit.

## Options

- **Bulb to monitor** — the Hue bulb with Zigbee Connectivity enabled on the switched circuit.
- **Name of Light Switch** — optional. Used as the alert title, as *"<name> Is OFF!"*. Leave it blank and it uses the bulb's own name plus "Switch", e.g. *"Guest Bath 1 Switch"*. Type your own if you'd rather the notification to use something else, like "Downstairs Bathroom".
- **Canary Bulb** — optional, but highly recommended. A canary bulb is a Hue bulb elsewhere in the house, on a different switch that isn't normally turned off. If the bridge, your network, or the whole house's power drops, every bulb goes unreachable at once; so if the canary bulb is also not answering, the automation does not alert, because it's more likely a bridge or power problem rather than a wall switch. Leave empty to skip the check.
- **Confirm for** — how long connectivity must stay broken before you're told. Default 2 minutes.
- **Who to notify** — one or more `notify` entities (the companion app on each phone). Add as many as you like.
- **Keep reminding me** — off gives one alert per switch-off; on nags until the switch goes back on. I'd recommend on, because the first alert usually lands when you can't act on it.
- **Remind me every** — the reminder interval.
- **Only remind me after** / **And stop reminding me at** — the reminder window, hourly between 08:00 and midnight by default. Outside the window you still get the *first* alert immediately — you just don't get nagged again until the window reopens. Set the end time equal to or earlier than the start to mean "until midnight".

The reminder loop ends the moment the bulbs answer the bridge again — I measured it exiting 13 milliseconds after connectivity came back. There's deliberately no notification when the switch goes back on: you flipped it, you know.

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
