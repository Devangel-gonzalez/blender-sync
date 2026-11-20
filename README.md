<div align="right">

**English** · [Español](README.es.md)

</div>

<div align="center">

# True Level: Playblast

**Zero-render live playblast. Animate while it plays.**

[![License](https://img.shields.io/badge/license-GPL--3.0--or--later-blue)](LICENSE)
[![Blender](https://img.shields.io/badge/Blender-4.5%2B-orange)](https://www.blender.org/)
[![Version](https://img.shields.io/badge/version-1.0.0-brightgreen)](blender_manifest.toml)
[![Platforms](https://img.shields.io/badge/Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)](blender_manifest.toml)

<!-- MEDIA 01 · replace the blockquote with the <img> line below once the file exists -->

> 🎬 **`docs/media/01-hero.gif`** — Two monitors. Left: hands posing a character. Right: the same character playing on a loop, never stopping, the edits landing in the running animation.

<!-- <img src="docs/media/01-hero.gif" alt="Editing on one monitor while the animation keeps playing on the second" width="900"> -->

</div>

---

## Stop pressing Play

Pose a frame. Hit play. Stop. Scrub back. Adjust. Hit play again.

Hundreds of times a day. **Every time you want to check your work, you have to stop making it.**

Playblast keeps your animation running while you edit it. A second viewport plays your scene on a loop and never stops — your changes appear inside that loop instantly, in context, in motion.

Second monitor, second Blender instance, same machine. **No render. No export. No stopping.**

And when someone else needs to see it, they connect from their own machine and watch the same live loop.

<div align="center">

|                                        |                                       |
| :------------------------------------: | :-----------------------------------: |
|              🧍 **Solo**               |          👥 **With others**           |
| A live review monitor that never stops | Live review sessions from any machine |
|    One machine · no network needed     |          Same local network           |

</div>

---

## 🧍 Solo — your own live review monitor

**This is the one that changes your day.** You do not need a second person, a studio, a network or a render farm. You need a second Blender window.

```
   YOUR MACHINE
   ┌──────────────────────────┐        ┌──────────────────────────┐
   │  Blender #1   ·   HOST   │  live  │  Blender #2   ·  VIEWER  │
   │                          │ ─────► │                          │
   │  you pose and key here   │  pose  │  plays on a loop,        │
   │                          │        │  never stops             │
   └──────────────────────────┘        └──────────────────────────┘
          monitor 1                           monitor 2
```

Set it up once:

1. **Blender #1** — open your animation file. Sidebar (`N`) ▸ **Multiuser** ▸ **Hosting** ▸ press **Host**, then **Init**.
2. **Blender #2** — open a new empty scene. Same tab ▸ select the **`localhost`** preset that ships with the add-on ▸ press **Connect**.
3. Drag Blender #2 to your second monitor, set the frame range you want to watch, and press **Play**.

Now go back to Blender #1 and animate. The loop keeps running. Every pose you touch shows up in it while it plays.

> Use the shipped **`localhost`** preset rather than creating your own — it is already configured for same-machine sessions.

<!-- MEDIA 02 -->

> 🎬 **`docs/media/02-solo-loop.gif`** — The whole idea in one shot: a bone is adjusted in the working window and the change appears in the looping playback window without anyone pressing stop.

<!-- <p align="center"><img src="docs/media/02-solo-loop.gif" alt="A pose edit appearing instantly inside a running playback loop" width="900"></p> -->

**No second monitor?** Put the two windows side by side, or use a second workspace. The loop is the point, not the hardware.

---

## 👥 With others — live review, no renders

Same add-on, other machines. A director, a rigger, a classmate, a client — anyone can watch the animation as it is being made, at full viewport quality, from their own Blender.

```
   HOST  ──►  CLIENT 1  ·  CLIENT 2  ·  CLIENT N
   animates   each on their own machine, each with their own camera and timeline
```

1. **Host** — same as above: **Host**, then **Init**. Share your local IP (`192.168.x.x`) and port `5555`.
2. **Client** — sidebar ▸ **Multiuser** ▸ **`+`** ▸ enter a name, the Host's IP and the port ▸ select it ▸ **Connect**.

No accounts. No router configuration. No sending the `.blend` over Discord or Drive.

> ### ⚠️ Clients: open an empty or disposable file
>
> **In the current build, joining over the network clears the Client's local scene.** Never connect with your only copy of work in progress. A connection dialog with _Combine / Replace / New scene_ is coming.
> This does **not** affect the solo `localhost` setup above.

<!-- MEDIA 03 -->

> 🎬 **`docs/media/03-live-review.gif`** — Two machines side by side. The animator works; the reviewer orbits around the character and scrubs their own timeline, while the animation keeps coming through.

<!-- <p align="center"><img src="docs/media/03-live-review.gif" alt="A reviewer on another machine orbiting and scrubbing while the animator works" width="900"></p> -->

---

## What it feels like

- 🎬 **The loop never stops.** The viewer's playback is completely independent. Checking your work costs you nothing.
- 🦴 **Drags look like drags.** Pose travels as raw animation channels at up to **60 updates per second** — not a slideshow of snapshots.
- 🎥 **Your camera, your framing.** Orbit, pan and zoom freely. Nobody yanks your view around.
- ⏱ **Your timeline.** Scrub anywhere, loop any range. Watch frame 12 on repeat while the animator works on frame 40.
- 🖱 **Click anything.** Select the rig, pick bones, inspect curves. The viewer is a real Blender session, not a video feed.
- 🔒 **Your work is safe.** Only the Host sends data. A reviewer can look at everything and break nothing.
- 📦 **Nothing to install.** Everything is bundled. No pip, no dependencies, no setup.

<!-- MEDIA 04 -->

> 🎬 **`docs/media/04-pose-live.gif`** — Close-up: one shoulder rotated on the Host, following in real time on the viewer, timeline running on both.

<!-- <p align="center"><img src="docs/media/04-pose-live.gif" alt="A bone drag reproduced live on the viewer" width="900"></p> -->

### Following someone on purpose

In **Online users**, pick a person and use 📷 to match their view or 🕐 to follow their playhead. Both are opt-in, both are reversible — `Esc` or right-click gets you out. Neither happens on its own.

### Presence overlay

`Overlays ▸ Multi-user` in the viewport header shows other people's cameras, mode and selection. Handy for knowing who is looking at what.

---

## Install

1. Download the `.zip` from [Releases](../../releases).
2. Blender ▸ `Edit ▸ Preferences ▸ Get Extensions ▸ ▾ ▸ Install from Disk` ▸ pick the `.zip`.
3. Enable **True Level: Playblast**.
4. Turn on `Preferences ▸ System ▸ Allow Online Access`. The add-on reminds you on first launch if it is off.

The panel lives in the 3D Viewport sidebar (`N`) under the **Multiuser** tab.

|               |                                             |
| ------------- | ------------------------------------------- |
| **Blender**   | 4.5 or newer · tested on 5.2.2              |
| **Platforms** | Windows x64 · macOS x64 and ARM · Linux x64 |
| **Sessions**  | Same machine, or same local network         |

---

## Options

**General Settings ▸ Replication**, mirrored as three icons in the **Synchronize** panel during a session.

<!-- PUBLISH CHECK · this table describes R1 as specified. Two values depend on
     approved changes that are NOT in the code yet:
       · Synchronize active camera = Off   → needs D-39 (code default is still True)
       · Apply delay = 0                   → needs D-48 (code default is still 1)
     If you publish before those land, change those two rows to the current
     defaults. The network scene-purge warning IS accurate for the current build;
     remove it once D-40a / D-42 ship. See RELEASE-CHECKLIST.md. -->

| Option                      | Default | What it does                                                          |
| --------------------------- | ------- | --------------------------------------------------------------------- |
| **Pose Live Sync** 🦴       | **On**  | The live pose channel. The heart of the add-on.                       |
| Synchronize active camera   | Off     | Shares which camera is active. **Does not force your viewport view.** |
| Synchronize render settings | Off     | Syncs Eevee and Cycles settings.                                      |
| Edit mode updates           | Off     | Sends mesh edits in Edit Mode. Costly on heavy meshes.                |
| Apply delay                 | `0`     | How often incoming data is applied. Leave at `0`.                     |

<!-- MEDIA 05 -->

> 📷 **`docs/media/05-options.png`** — Still screenshot: the Replication toggles and the Synchronize icon row.

<!-- <p align="center"><img src="docs/media/05-options.png" alt="The replication options and the synchronize icon row" width="420"></p> -->

**Frame range:** the viewer inherits the Host's range on connect, then it is theirs to change with vanilla Blender. It only resets if the Host later changes their own.

---

## Roadmap

<pre>
True Level Roadmap
│
├── True Level: Playblast ──┬── <code>Phantom Keyframe Update</code>
│                           └── <code>Datalink TWS Update</code>
</pre>

**`Phantom Keyframe Update`** — full control over the temporary keys the live channel creates while a pose is being dragged. A **Phantom Keyframe Mode** you can switch on and off, buttons to keep or delete them from the Host's Dope Sheet, Host phantoms that survive saving the `.blend`, and auto-key as a supported capability. In 1.0 the goal is simply _no phantoms_; this update turns them into a tool.

**`Datalink TWS Update`** — sessions **across the internet**, no accounts, no router configuration, join with a session code. The name is a nod to combat-flight-sim datalinks and track-while-scan: share the picture, keep everyone in sync, nobody has to stop flying.

---

## Good to know

Straight answers rather than fine print:

- **Two instances means two copies of the scene in memory.** On a heavy rig, budget RAM accordingly. The solo setup is still far cheaper than rendering a playblast.
- **Joining over the network clears the Client's scene** in this build. The solo `localhost` setup is unaffected. A proper dialog is coming.
- **Host visibility and restriction toggles are not preserved on the viewer yet.** Being fixed.
- **One armature streams at a time.** Simultaneous multi-armature selection is in progress.
- **The `Live Sync` per-datablock toggle is inherited and unsupported in 1.0.** Left in place so nothing changes for existing users, but not maintained. You do not need it — keyframes arrive through the normal path with it off.
- **Brief flicker** is possible when you confirm a transform, key it and deselect in very quick succession. Known, accepted for 1.0.
- **Third-party bone physics** compose with the live pose rather than replacing it. Flicker on the exact bone being dragged while physics is active on it also happens without this add-on.
- **Auto-key is not supported in 1.0.**
- **Internet sessions** are not part of 1.0. VPN workarounds may work but are unsupported.

---

<details>
<summary><b>📋 Media checklist — what to record for the slots above</b></summary>

Drop the files in `docs/media/`. Each slot is a blockquote with a commented `<img>` line right below it: delete the blockquote, uncomment the line. Remove this section when everything is filled in.

| Slot | File                 | What to capture                                                                                     | Length |
| ---- | -------------------- | --------------------------------------------------------------------------------------------------- | ------ |
| 01   | `01-hero.gif`        | Two monitors: editing on one, the loop running on the other. **Lead with the solo case.**           | 8–12 s |
| 02   | `02-solo-loop.gif`   | A bone adjusted in the working window, the change landing in the running loop. Nobody presses stop. | 5–8 s  |
| 03   | `03-live-review.gif` | Two machines: animator working, reviewer orbiting and scrubbing independently.                      | 6–10 s |
| 04   | `04-pose-live.gif`   | Close-up of one bone dragged on the Host, following live on the viewer.                             | 4–6 s  |
| 05   | `05-options.png`     | Still: Replication toggles + the Synchronize icon row.                                              | —      |

Optional: `00-banner.png`, a wide title banner above the badges.

Keep GIFs under ~5 MB, 900 px wide, cropped to the viewport, looping cleanly. If one gets too heavy, an `.mp4` uploaded to GitHub renders as a native player and is much smaller — same slot.

</details>

---

## Credits and license

**True Level: Playblast** is maintained by **Soilent** ([@Devangel-gonzalez](https://github.com/Devangel-gonzalez)).

Built on **[Multi-User](https://gitlab.com/slumber/multi-user)** for Blender by **Swann Martinez (Slumber)**, Copyright © 2019–2026 ([extension page](https://extensions.blender.org/add-ons/multi-user/) · [docs](https://slumber.gitlab.io/multi-user/index.html)). The replication layer, the session model and the presence overlay come from that work, and this project would not exist without it. Thank you.

**GPL-3.0-or-later** — see [LICENSE](LICENSE) for the full text and both copyright notices.

```
Copyright (C) 2026 Soilent (github.com/Devangel-gonzalez)
Based on "Multi-User" for Blender — Copyright (C) 2019-2026 Swann Martinez (Slumber)
```

## Issues and feedback

Bug reports and ideas welcome in [Issues](../../issues). Include your Blender version, whether you were Host or Viewer, and what you did just before it happened. A recording of both viewports beats a thousand lines of log.
