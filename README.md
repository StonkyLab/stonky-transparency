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
