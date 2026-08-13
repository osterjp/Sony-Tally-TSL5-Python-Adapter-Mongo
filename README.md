[README.md](https://github.com/user-attachments/files/31037785/README.md)[Upl# HELIX TSL 5.0 Tally-to-MongoDB Adapter

A small on-premises Python service that receives TSL UMD V5.0 over UDP, qualifies **final Program tally** separately from Preview, timestamps Program ON/OFF edges, stores them in a crash-safe local SQLite outbox, and writes them idempotently to MongoDB Atlas.

## Data path

```text
Sony switcher
    -> Evertz MAGNUM-TALLY
    -> duplicate TSL UMD V5.0 UDP feed
    -> this Docker service
    -> MongoDB Atlas tally_events + on_air_intervals
```

The service does not inspect multiviewer pixels. It consumes the underlying TSL tally message that drives the tally indication.

## What it records

Only two qualified edge events enter `tally_events`:

- `PROGRAM_ON`: configured Program state changed from false to true.
- `PROGRAM_OFF`: configured Program state changed from true to false.

A Preview-only state does not create an on-air event. Repeated TSL refresh packets do not create duplicates. The same ON/OFF pair also creates and closes one document in `on_air_intervals`.

## Critical assumptions

1. Evertz is configured to send standard TSL UMD V5.0 UDP to the Docker host.
2. The mapping file correctly identifies which TSL lamp/text field and color represent Sony **final Program**, not Preview or look-ahead tally.
3. Each configured display index represents a stable logical source. If the multiview tile dynamically changes sources, integrate the Evertz route event or add source-resolution logic before production use.
4. TSL V5.0 does not contain an event timestamp. This service timestamps the first changed state it receives. The host clock must therefore be synchronized to the facility PTP/time domain, or the clock provider must be replaced with an LTC/ATC-aware source.
5. Any value in `TALLY_LATENCY_CORRECTION_MS` must be measured in the actual Sony-to-Evertz path; do not guess it.

## Quick local test

```bash
cp .env.example .env
docker compose up --build -d
```

The example starts with `SINK_MODE=stdout`, so no Atlas connection is required.

Send a baseline OFF packet:

```bash
python3 scripts/send_test_packet.py --color OFF
```

Send Preview only; this must not create a Program event:

```bash
python3 scripts/send_test_packet.py --color GREEN
```

Send Program ON and OFF:

```bash
python3 scripts/send_test_packet.py --color RED
python3 scripts/send_test_packet.py --color OFF
```

Inspect logs and health:

```bash
docker compose logs -f tally-adapter
curl http://127.0.0.1:8080/status
```

Expected result: one `PROGRAM_ON`, one `PROGRAM_OFF`, and no event for GREEN Preview.

## Configure the Evertz feed

Configure MAGNUM-TALLY or the associated Evertz UMD output to send a duplicate TSL UMD V5.0 **unicast UDP** stream to:

```text
Destination IP: IP address of the Docker host
Destination port: 8900 by default
Protocol: TSL UMD V5.0
```

Unicast is the simplest container deployment. Multicast is supported by the service, but Docker multicast behavior depends on the host network and may require `network_mode: host`.

Set `TSL_ALLOWED_SOURCES` to the Evertz sender IP after testing:

```dotenv
TSL_ALLOWED_SOURCES=10.20.30.40
```

## Configure Program versus Preview

Edit `config/mappings.json`. The included example assumes:

```json
"program_tally": {
  "selector": "text",
  "states": ["RED"]
},
"preview_tally": {
  "selector": "text",
  "states": ["GREEN"]
}
```

Valid selectors are `right`, `text`, and `left`. Valid states are `OFF`, `RED`, `GREEN`, and `AMBER`.

Do not rely on the color name alone. Confirm the actual Evertz/Sony mapping in PCR 6 and set the selector and state that mean **contributing to the configured final Program destination**.

The display mapping must also identify the source and path:

```json
{
  "screen": 0,
  "display_index": 27,
  "source_id": "GFX_1",
  "switcher_id": "PCR6_SONY",
  "destination_id": "PCR6_PGM_1",
  "destination_role": "FINAL_PROGRAM",
  "layer_id": "KEY_2"
}
```

`destination_role` is deliberately restricted to `FINAL_PROGRAM` in this adapter.

## Connect to MongoDB Atlas

Update `.env`:

```dotenv
SINK_MODE=mongodb
MONGODB_URI=mongodb+srv://USER:PASSWORD@CLUSTER.mongodb.net/?retryWrites=true&w=majority
MONGODB_DATABASE=helix
```

Atlas must allow network access from the Docker host or its egress address. Use a least-privilege database user that can update the two collections and create indexes when `MONGODB_ENSURE_INDEXES=true`.

The adapter uses:

- TLS certificate verification.
- MongoDB Stable API v1.
- Majority write concern and retryable writes.
- Event `_id` upserts for idempotency.
- A SQLite write-ahead outbox for WAN or Atlas outages.

No MongoDB credential is built into the image. Supply it at runtime through the deployment's secret-management mechanism.

## Stored event example

```json
{
  "_id": "uuid",
  "event_type": "PROGRAM_ON",
  "air_scope": "PRODUCTION_PROGRAM",
  "source": {
    "source_id": "GFX_1",
    "source_label": "Graphics Channel 1"
  },
  "path": {
    "switcher_id": "PCR6_SONY",
    "destination_id": "PCR6_PGM_1",
    "destination_role": "FINAL_PROGRAM",
    "layer_id": "KEY_2"
  },
  "tally": {
    "protocol": "TSL_UMD_V5_0",
    "screen": 0,
    "display_index": 27,
    "program_state": true,
    "preview_state": false,
    "text_tally": {
      "code": 1,
      "color": "RED"
    }
  },
  "time": {
    "observed_at_utc": "2026-08-13T18:42:15.347000Z",
    "effective_at_utc": "2026-08-13T18:42:15.347000Z",
    "timestamp_basis": "EDGE_RECEIPT",
    "clock_locked": true,
    "house_timecode": {
      "value": "14:42:15;10",
      "rate_num": 30000,
      "rate_den": 1001,
      "drop_frame": true
    }
  }
}
```

## Timing behavior

The timestamp is taken immediately when the UDP datagram reaches the adapter process.

- `observed_at_*`: actual local receipt time.
- `effective_at_*`: receipt time minus the configured measured latency correction.
- `house_timecode`: frame label calculated from `effective_at_ns` and the configured frame rate/time zone.
- `clock_locked`: read from `CLOCK_LOCK_STATUS_FILE` when configured; otherwise uses `CLOCK_LOCKED_DEFAULT`.

For exact LTC-derived timecode, replace `TimecodeClock` with a facility-approved LTC/ATC reader. Do not label the clock as locked merely because the host has ordinary NTP.

## Restart and outage behavior

The SQLite volume stores both the last Program state and unsent events.

- Repeated Program packets after restart do not create duplicate ON records.
- If Atlas is down, events remain locally queued and are retried in order.
- If the service starts while Program is already ON, the default is to establish a baseline without inventing an ON time. Set `EMIT_INITIAL_PROGRAM_ON=true` only when an observed-start record is acceptable.
- Do not delete the SQLite state while an interval is open.
- Changing a screen/index mapping causes the service to reject that display until its prior interval is resolved and the local state is deliberately reset.

## Health endpoints

```text
GET /healthz  process liveness
GET /readyz   listener readiness
GET /status   counters, outbox depth, last packet, and last error
```

## Run tests without Docker

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements-dev.txt
pytest -q
```

The included tests cover TSL V5 decoding, multiple messages, malformed PBC handling, 29.97 drop-frame boundaries, Program edge suppression, initial-state behavior, and outbox ordering.

## Production acceptance test

Before using the data as proof of air, validate at minimum:

1. Green/Preview creates no on-air record.
2. Sony final Program creates one `PROGRAM_ON`.
3. Leaving final Program creates one `PROGRAM_OFF` with the same interval ID.
4. Repeated Evertz refresh packets create no duplicates.
5. A keyer contribution is recognized correctly.
6. A non-contributing M/E does not assert final Program.
7. The configured UMD index and source identity remain correct after router changes.
8. Host PTP lock and Sony-to-MAGNUM-to-adapter latency are measured and documented.

This code is implementation-ready but must be validated with packet captures from the installed Evertz MAGNUM release and the actual Sony tally configuration before production certification.
oading README.md…]()
