# go-wyze: the design

Research into Wyze cameras and the Wyze Garage Door Controller, done on 2026-11-15 from
public sources, to decide what a Go module for them would be. Nothing is built and
nothing here was run against a device. The decisions at the end are open.

In short: the two are different cases. A camera's video can be had on the LAN: several
models stream directly once their keys have been fetched from Wyze's cloud, and Wyze now
ships RTSP on some. The garage door controller has no local route at all: it is driven
through Wyze's cloud, over the app's undocumented API.

## The devices

- **Cameras** stream over a peer-to-peer protocol, ThroughTek's, under DTLS. A client
  needs the camera's P2P id and key, which Wyze's cloud gives to the account. Some newer
  models use another protocol
  ([go2rtc's Wyze notes](https://github.com/AlexxIT/go2rtc)).
- **The Garage Door Controller** is an accessory of the Wyze Cam v3, not a device of its
  own: a relay to the opener, with the camera reading a marker on the door to tell open
  from closed. It works only with that camera on Wyze's own firmware.

## Ways in: video

### Peer to peer, on the LAN

From go2rtc's notes on its Wyze source, `internal/wyze/README.md`:

- "Internet access is only needed when loading cameras from your account. After that,
  all streaming is local P2P", and "Connection to the camera is local only (direct P2P to
  camera IP)".
- It needs firmware with DTLS. The stream is addressed by the camera's address, its P2P
  id, its key, its MAC and its model.
- Its table of models marks the Cam OG, OG Telephoto and Pan v4 as using the other
  protocol, which it does not support.

go2rtc is Go, and its Wyze and ThroughTek packages can be imported. Wyze can change the
encryption in a firmware update: [one bridge project](https://github.com/IDisposable/docker-wyze-bridge)
reports that Cam v4 firmware from 2025 blocks this route.

### RTSP in Wyze's firmware (local, official)

Wyze's staff describe RTSP for the Cam v3 and Pan v3
([its forum thread](https://forums.wyze.com/t/rtsp-for-v3-and-pan-v3/338351)), set in the
app with a user name and password. Reports put it in production firmware in 2026, absent
on the v3 Pro, in beta on the v4.

### Replacement firmware

[Thingino](https://github.com/themactep/thingino-firmware) replaces Wyze's firmware on
several models and gives RTSP. It takes the camera out of Wyze's cloud, and its notes list
the garage door controller as not supported.

## Ways in: the garage door

- **Local: none found**, and replacement firmware drops the accessory.
- **Wyze's cloud, as the app uses it.** Wyze issues an API key and key id to an account
  from its developer console; they are login credentials, and the endpoints are not
  documented. In the source of [wyze-api](https://github.com/jfarmer08/wyze-api), the
  login carries the key and key id as headers, and the door is moved by an action named
  `garage_door_trigger` sent for the camera. It is a toggle: a client reads the door's
  state, a property of the camera, before sending it.
- The door's state is the camera's reading of the marker, not a switch.
- A login protected by a code sent by text or mail cannot run unattended; one by an
  authenticator app can.

## The ways in, compared

| | Video, peer to peer | Video, Wyze's RTSP | Garage door |
|---|---|---|---|
| Supported by Wyze | no | yes, on some models | no |
| Local at run time | yes | yes | no |
| Needs | one fetch of the camera's keys from the account | the setting in the app | the account, an API key and key id |
| Risk | a firmware update can close it | patchy across models | an undocumented API; the action is a toggle |

## What this module would be

Not decided, and it is two modules in one name:

1. **Video.** RTSP where the camera has it needs no module: any player reads it. Where
   it does not, go2rtc's packages, with one fetch of the keys kept as configuration.
2. **The garage door.** A client of the app's API: the login, the camera's properties,
   the trigger guarded by the door's state. Small, and on an interface that can change
   without notice. A wired board on the opener, as go-myq describes, is the local
   alternative to the accessory.

No Go client of Wyze's device API with the garage door was found.

## Open questions

- The model and firmware of each camera.
- Whether the camera behind the garage door controller is a Cam v3 on Wyze's firmware,
  as the accessory requires.
- How the Wyze account's second factor is set: an authenticator app, or codes by text or
  mail.
- Whether video is wanted from this module at all, or only the door.

## Not verified

- Nothing above was observed on a device.
- How the controller attaches to the camera, and Wyze's own statement of what it works
  with: its support pages refused to be read.
- The property that carries the door's state, read in one plugin's source and not in
  wyze-api's.
- Which firmware versions carry RTSP: from forum posts seen as search excerpts.
- The properties for a camera's power, siren and floodlight.

## Sources

- [go2rtc](https://github.com/AlexxIT/go2rtc): local streaming, what it needs from the
  cloud, the models it covers.
- [docker-wyze-bridge](https://github.com/IDisposable/docker-wyze-bridge): a Go bridge;
  firmware that closes the local route.
- [RTSP for v3 and Pan v3](https://forums.wyze.com/t/rtsp-for-v3-and-pan-v3/338351):
  Wyze's staff on RTSP.
- [Thingino](https://github.com/themactep/thingino-firmware): replacement firmware and
  what it drops.
- [wyze-api](https://github.com/jfarmer08/wyze-api): the app's API, the login, the
  garage door's action.
- [homebridge-wyze-smart-home](https://github.com/jfarmer08/homebridge-wyze-smart-home):
  the door's state as a camera property.
- [wyze-sdk](https://github.com/shauntarves/wyze-sdk): the app's API in Python, and its
  second-factor handling.
