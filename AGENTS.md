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
