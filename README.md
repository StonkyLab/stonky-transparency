# Stonky transparency

Every decision of Stonky's live strategies is recorded as a signal - the full
book the strategy intends to hold, at the moment it decides - and appended to
the transparency log of its portfolio (Tessera; checkpoints are C2SP signed
notes). The log is append-only: an entry cannot be changed or removed without
changing every checkpoint that came after it.

The signals themselves are published a month later. Until then this repository
fixes the state of each log once a day:

    checkpoints/<portfolio>/<YYYY-MM-DD>.checkpoint       the log's checkpoint at 23:00 UTC:
                                                          origin, number of entries, root hash, signature
    checkpoints/<portfolio>/<YYYY-MM-DD>.checkpoint.ots   its OpenTimestamps proof

A checkpoint commits to every signal in the log without revealing any of them.
Its OpenTimestamps proof anchors it in the Bitcoin blockchain within hours, so
the day a checkpoint existed rests neither on our word nor on this repository's
history. Once a month's signals are published, anyone can check that they are
exactly the entries these checkpoints committed to, on the day they were made.

## Logs

| Portfolio | Origin | Verifier key |
|---|---|---|
| Stocks | `stonky.cz/signals/stocks` | `stonky.cz/signals/stocks+3e2485ba+AYsX8zyIzaVDa/sbBUaYEsEnxvI1u0v1LjZuPM7PEWPI` |

## Verifying a timestamp

Upload a `.checkpoint` file with its `.ots` proof at https://opentimestamps.org,
or use the client with a Bitcoin node:

    pip install opentimestamps-client
    ots verify checkpoints/stocks/2026-10-14.checkpoint.ots

A proof made the same day can still be pending; the next daily runs add the
Bitcoin attestation.

## Failed records

Days on which a log's checkpoint did not get its proof on its own day. A proof
made later does not show that the day's signals existed on that day, so every
such day is listed here.

| Date | Concerns | Reason |
|---|---|---|
| 2026-11-11 | Stocks: `checkpoints/stocks/2026-11-11.checkpoint`, the 5 entries the log gained that day (its size went from 100 to 105) | The OpenTimestamps stamp at 23:00 UTC failed and the job did not try again that day. The proof was made by the next run, on 2026-11-12 at 23:00 UTC: it shows that these entries existed by then, not on the 11th. Since 2026-11-13 the job repeats the stamp until it succeeds. |
