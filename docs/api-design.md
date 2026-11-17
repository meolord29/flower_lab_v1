# API design

The cabinet is the client. The main server is the collector. All writes are POST. The cabinet does not poll the server to decide a dose.

Base path: `/v1`. Bodies are JSON. Times are ISO-8601 UTC.

## Identity

Every request carries the cabinet id in the path and a bearer token.

```
Authorization: Bearer <token>
```

The token is provisioned on the Pi. A missing or wrong token is `401`. The cabinet still runs the loop.

## Idempotency

Each reading, dose, and fault has an `id` generated on the Pi. The server treats a repeated `id` as the same row and returns the original success. Retries after a timeout must reuse that id.

## POST /v1/cabinets/{cabinet_id}/batch

The normal call. One body, any mix of new records. An empty list is valid and is how the cabinet can check the link.

```json
{
  "fw": "0.1.0",
  "readings": [],
  "doses": [],
  "faults": []
}
```

Shapes are the models in [data-models.md](data-models.md).

Success: `202 Accepted`

```json
{ "accepted": 3, "duplicate": 1 }
```

`accepted` counts new rows. `duplicate` counts ids already stored. Both are success for the client, which may drop those ids from its queue.

Errors:

| Status | When | Client |
| --- | --- | --- |
| `400` | Body is not the contract | Drop the body. Do not retry it |
| `401` | Token rejected | Keep the queue. Do not invent a new token |
| `404` | Unknown `cabinet_id` | Keep the queue |
| `413` | Body too large | Split the batch and retry |
| `5xx`, timeout | Server or network | Keep the queue. Retry with the same ids |

The cabinet does not wait on this call inside a pump pulse. The loop and the sender are separate. A full queue drops the oldest reading before it drops a dose or a fault.

## GET /v1/cabinets/{cabinet_id}/aggregates

Optional. For a person, not for the control loop.

Query: `grain=hour|day`, `from`, `to` as UTC timestamps. Default grain is `day`. Default window is the last 7 days.

```json
{
  "cabinet_id": "cab-1",
  "grain": "day",
  "buckets": [
    {
      "bucket": "2026-11-17T00:00:00Z",
      "moisture_avg": 18420,
      "temperature_min": 19.5,
      "temperature_max": 26.1,
      "light_on_seconds": 50400,
      "water_seconds": 18,
      "a_seconds": 2,
      "b_seconds": 0,
      "npk_last": { "n": 41, "p": 22, "k": 35 },
      "vision_counts": {}
    }
  ]
}
```

`vision_counts` is `{}` until the camera exists.

## What is not an endpoint

- No remote "pump now". A person at the cabinet can override locally. The server cannot open a MOSFET.
- No setpoint PUT in this version. Setpoints live on the Pi.
- No image upload. Later vision posts a label, a confidence, and a time. Frames stay on the cabinet unless a later revision says otherwise.

## Cadence

| Data | When it is posted |
| --- | --- |
| Moisture, temperature, light, float | Each control cycle, or at least once a minute |
| NPK | Each successful read, a few times a day |
| Dose | When the pulse starts |
| Fault | When it starts, and when it clears (`detail` says `cleared`) |
| Vision | Later. Each inference, a frame every few minutes |
