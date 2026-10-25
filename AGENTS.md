# petlibro fork: motion events + camera for Granary Smart Camera Feeder (PLAF203)

Fork of jjjonesjr33/petlibro (remote `upstream`; ours is `origin` = luwi8011/petlibro, public because it is a fork).
Goal: expose motion detection in Home Assistant (event/binary_sensor, image if alert has a snapshot), then a camera via go2rtc + TUTK.

Rules
- One Petlibro login per account: never run scripts and the HA integration at the same time.
- Credentials only in `.env` (gitignored; template `.env.example`). Raw captures go in `captures/` (gitignored), they may hold tokens.
- Do not change HA config beyond installing the fork; ask first.
- Leave automation.poppy_feeder_alerts alone.
- Plan: 1 read api/coordinator, 2 debug-dump raw JSON (device info, events, alerts/message center), 3 find motion source,
  4 add entities, 5 automation (only after 4), 6 camera.

## Findings (2026-10-25)
- `/data/event/deviceEventsV2` stays empty on PLAF203 firmware 3.1.48; the old motion sensor never fired.
- Motion/sound alerts live in the app message center: `GET /device/msg/list` (latest ~10) and `GET /device/msg/page`
  (paged, ~773 rows). Entries have `eventKey` DETECTION_MOVE / DETECTION_SOUND, `timestamp` (ms), and the device serial
  inside the JSON string `param`. `imageUrl` is always null: no snapshot in the alert.
- `GET /device/msg/detail?id=` marks the message as read; the list endpoints do not.
- The Petlibro cloud rate-limits alerts (roughly one per few minutes), so detections are not instantaneous per-motion.
- Implemented: `api.device_messages`, `GranarySmartCameraFeeder.motion_detections/sound_detections`, binary sensors
  (on for 2 min after an alert), `event.py` event entities (fire once per new message, seeded at startup).
- `petlibro.debug_api_probe` (GET/POST any path, returns raw JSON) is temporary; remove before release.
- Remaining: automation (ask first), camera via TUTK/go2rtc (unproven; Kalay handshake unknown).

## Camera (2026-10-25)
- Firmware: update.granary_smart_camera_feeder_firmware shows 3.1.48 installed = latest, so no newer firmware exists.
- Stream works LAN-only via tannerln7/go2rtc release v1.9.14-petlibro.3 (binary `petlibro-go2rtc_linux_amd64`, sha256 verified).
  Stream URL: `petlibro://<feeder-ip>?uid=<camera_id>&quality=hd`; feeder = 192.168.0.5 (MAC 6c:d5:52:5f:18:88), UDP 32761.
- Test install in the dev container: `/config/.local/share/petlibro-go2rtc/` (go2rtc + go2rtc.yaml with UID, chmod 600, not in git;
  UID also in `.env` as CAMERA_UID). Start: `./go2rtc -config ./go2rtc.yaml`. Not persistent across container restarts.
- Verified: ffmpeg grab of `rtsp://127.0.0.1:8554/petlibro_feeder` gave a 1920x1080 frame in ~7 s cold, 3 grabs in a row OK.
  The reported 1-2 minute stall on 3.1.48 (go2rtc fork PR #1, ack_hole_timeout) only matters for long streams; snapshots are short.
- There is no camera-only HA add-on; Petlibro Local add-on (tannerln7/Petlibro-Home-Assistant) also takes over feeder control: do not use.
- Goal flow: successful feeding -> both persons away -> wait for next motion alert (Poppy at bowl) -> grab frame -> notify both phones.
