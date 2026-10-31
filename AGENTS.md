# petlibro fork: motion events + camera for Granary Smart Camera Feeder (PLAF203)

Fork of jjjonesjr33/petlibro (remote `upstream`; ours is `origin`, public because it is a fork).
Household-specific setup notes (IPs, entity names, automations, Portainer stack) live in `local/` (gitignored). Read
`local/SETUP-NOTES.md` first when working on this install.

Rules
- One Petlibro login per account: never run scripts and the HA integration at the same time.
- Credentials and device identifiers only in `.env` / `local/` (gitignored). Never commit the camera UID, device serial,
  tokens, or LAN addresses.
- Do not change HA config without asking the owner.

## Findings (PLAF203, firmware 3.1.48)
- `/data/event/deviceEventsV2` stays empty on this firmware, so the upstream motion/sound sensors never fire.
- Motion/sound alerts live in the app message center: `GET /device/msg/list` (latest ~10) and `GET /device/msg/page`
  (paged). Entries have `eventKey` DETECTION_MOVE / DETECTION_SOUND, `timestamp` (ms), and the device serial inside the
  JSON string `param`. `imageUrl` is always null: alerts carry no snapshot.
- `GET /device/msg/detail?id=` marks the message as read; the list endpoints do not.
- The Petlibro cloud rate-limits alerts (roughly one per few minutes).
- Implemented: `api.device_messages`, `GranarySmartCameraFeeder.motion_detections/sound_detections`, binary sensors
  (on for 2 min after an alert), `event.py` event entities (fire once per new message, seeded at startup).

## Camera
- Live video works LAN-only via tannerln7/go2rtc (`petlibro://<feeder-ip>?uid=<camera_id>&quality=hd`, UDP 32761), then HA
  Generic Camera on the go2rtc RTSP output. `camera_id` is exposed as an attribute on the Wi-Fi SSID sensor.
- No newer firmware than 3.1.48 as of 2026-10. The reported 1-2 min stall (go2rtc fork PR #1) did not show in a 3 min soak.
- HA `camera.snapshot` times out at 10 s and fails ~1 in 3 cold grabs; a failed grab leaves a 0-byte file and a
  half-started stream can give a 640x360 frame. Verify file size before using a snapshot.
- iOS notification attachments: do not add a query string to `/media/local/...` URLs (404 in the app).
- HA 2026.10 snapshots of Generic Cameras go through HA's built-in go2rtc (`ffmpeg:` + stream source). HA caches go2rtc's
  `/api/schemes` once at startup; go2rtc opens its API before the ffmpeg module registers, so the cache can miss `ffmpeg`
  and every snapshot/live view fails with "Stream source is not supported by go2rtc" until go2rtc is reloaded
  (home-assistant/core#186806, fix PR #186969). Reloading the go2rtc config entry clears it.
