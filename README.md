# Stonky transparency

Every decision of Stonky's live strategies is recorded as a signal - the full
book the strategy intends to hold, at the moment it decides - and appended to
the transparency log of its portfolio on its trading venue (Tessera; checkpoints
are C2SP signed notes). The log is append-only: an entry cannot be changed or removed without
changing every checkpoint that came after it.

The signals themselves are published a month later. Until then this repository
fixes the state of each log within about two minutes of every new entry (within about a
minute until 2026-11-22 20:18 UTC), and its state at the end of each day:

    checkpoints/<log>/<YYYY-MM-DD>/<HH-MM-SS>.checkpoint   a new state of the log, read at that time
                                                           (UTC): origin, number of entries, root
                                                           hash, signature, and since 2026-11-22
                                                           the witnesses' cosignatures (below)
    checkpoints/<log>/<YYYY-MM-DD>.checkpoint              the log's checkpoint at the end of the day
                                                           (stocks at 23:00 UTC, crypto at 23:55 UTC)
    <checkpoint>.ots                                       its OpenTimestamps proof

A checkpoint commits to every signal in the log without revealing any of them.
Its OpenTimestamps proof anchors it in the Bitcoin blockchain: that the
checkpoint existed no later than its Bitcoin block, a few hours after it was
made, rests neither on our word nor on this repository's history. Once a month's
signals are published, anyone can check that they are exactly the entries these
checkpoints committed to.

## Logs

| Log | Portfolio, venue | Origin | Verifier key |
|---|---|---|---|
| `stocks` | Stocks, Interactive Brokers | `stonky.cz/signals/stocks` | `stonky.cz/signals/stocks+3e2485ba+AYsX8zyIzaVDa/sbBUaYEsEnxvI1u0v1LjZuPM7PEWPI` |
| `crypto-bybit` | Crypto, Bybit | `stonky.cz/signals/crypto/bybit` | `stonky.cz/signals/crypto/bybit+70fdcb1f+AQzGUUPfiCWD+S7rGM0WeqmT9+0DyZfO76HGc1oaI3oH` |
| `crypto-kraken` | Crypto, Kraken | `stonky.cz/signals/crypto/kraken` | `stonky.cz/signals/crypto/kraken+6a8b43b2+ATcbu3hXX+SAC+qkILoNklzZiVzuyYU+igKRf/PWygGp` |
| `crypto-okx` | Crypto, OKX | `stonky.cz/signals/crypto/okx` | `stonky.cz/signals/crypto/okx+1d2cc6b7+AXjwLFH8SVUEZNhsyIVW+1GrJSxMl3HeiqXd3IamAz9g` |

The stock log runs since 2026-10-14, the crypto logs since 2026-11-14. Until 2026-11-16 at
12:53 UTC each log was anchored once a day, at the end of the day; every new state since. The live
crypto track records start on 2026-11-17, the first whole day on which every new state of their
logs was anchored within about a minute: the crypto logs' entries of 2026-11-14 to 2026-11-16 stay
in the logs, outside those records.

## Witnesses

Since 2026-11-22 20:18 UTC the four logs request cosignatures from the independent witnesses below
before publishing a checkpoint, only when the log has grown and at most once a minute. A witness
cosigns a checkpoint only when it extends the last one the witness cosigned for that log, the
state it keeps (a consistency proof): as long as it keeps that state, it cosigns no rewritten or
forked history of the log. These witnesses serve witness-network.org's staging list
`log-list-100qps-40klogs.1` (https://staging.witness-network.org/log-list-100qps-40klogs.1), and a
staging witness may reset the state it keeps. A cosignature is a line after our signature in the
`.checkpoint` file, `— <witness> <base64>` (C2SP tlog-cosignature, with the time it was made),
checked with the witness's key below.

A checkpoint carries the verified cosignatures the writer received within 5 seconds and saved.
It lacks the cosignature of a witness that is unavailable or answers later. The writer saves
its signed checkpoint before contacting the witnesses and marks the submission unfinished until
their answers are in; after an interruption it sends that checkpoint to the witnesses again
before publishing it. A witness rejecting the history stops the log, also after a restart,
until the history is restored; a rejection that comes after the 5 seconds stops it two
checkpoints later. A witness that missed a checkpoint can cosign a later one. The witnesses,
with the page where each publishes its key:

| Operator | Verifier key | Published at |
|---|---|---|
| Geomys | `witness.navigli.sunlight.geomys.org+a3e00fe2+BNy/co4C1Hn1p+INwJrfUlgz7W55dSZReusH/GhUhJ/G` | https://witness.navigli.sunlight.geomys.org/witness.v0.json |
| IPng Networks | `krabbel.ct.ipng.ch+4b10dbd8+BO4sLOtnhktlqMCejDx9C5LDd+/xZxbmBb6BP9ZD2Def` | https://ipng.ch/s/ct/#witness (krabbel-ed25519.pub.vkey) |
| Markovian Protocol | `markovianprotocol.com/witness+41b8827f+BJKLl8rBYM0oUxSaWh9rjTldTprEpYj/SWVIGgacA9XC` | https://witness.markovianprotocol.com/about |
| Mullvad VPN | `witness.stagemole.eu+67f7aea0+BEqSG3yu9YrmcM3BHvQYTxwFj3uSWakQepafafpUqklv` | https://witness.stagemole.eu/about |
| TrustFabric | `staging.witness.transparency.goog/ring-any-bells+2e1a8dc9+BG5JTpLc3FJtwzgh1Uv+Qelz9qeOH2bfWjS1s0s+y4rL` | https://transparency.dev/witnesses |

A checkpoint made before 2026-11-22 20:18 UTC carries our signature alone, also when it is anchored again later as the day's checkpoint of a log that has not grown since: the stock log's until its first entry after that time.

## Verifying the cosignatures

On GitHub, use **Download raw** for both the `.checkpoint` and its `.ots` proof. This preserves
the original bytes even if your Git checkout would convert the checkpoint's line endings.

`verify.go` below checks a `.checkpoint` file: that it is a checkpoint of the log whose key is
given (that log's origin, a size and a 32-byte root, signed with that key), and that at least `<n>`
of the witness keys given cosign it, `<n>` being how many of them you require, from 0 to all.
The first signature line of each key given is checked, and one that does not verify fails the
whole file; a further line under the same name and key hash is neither checked nor counted. One
key given under two names, in either of its encodings, fails it too, and so does the log's key
given as a witness's (a cosignature does not sign the witness's name). With Go 1.26 or later:

    mkdir verify && cd verify && go mod init verify
    # verify.go: the program below
    go get golang.org/x/mod@v0.42.0 github.com/transparency-dev/formats@v0.0.0-20251017110053-404c0d5b696c
    go run . <checkpoint> <n> <log key> <witness key>...

The keys are the verifier keys of the tables above, a witness's also on its own page; e.g. for
the stock log with all five witnesses, two of them required:

    go run . ../checkpoints/stocks/<YYYY-MM-DD>.checkpoint 2 \
        stonky.cz/signals/stocks+3e2485ba+AYsX8zyIzaVDa/sbBUaYEsEnxvI1u0v1LjZuPM7PEWPI \
        witness.navigli.sunlight.geomys.org+a3e00fe2+BNy/co4C1Hn1p+INwJrfUlgz7W55dSZReusH/GhUhJ/G \
        krabbel.ct.ipng.ch+4b10dbd8+BO4sLOtnhktlqMCejDx9C5LDd+/xZxbmBb6BP9ZD2Def \
        markovianprotocol.com/witness+41b8827f+BJKLl8rBYM0oUxSaWh9rjTldTprEpYj/SWVIGgacA9XC \
        witness.stagemole.eu+67f7aea0+BEqSG3yu9YrmcM3BHvQYTxwFj3uSWakQepafafpUqklv \
        staging.witness.transparency.goog/ring-any-bells+2e1a8dc9+BG5JTpLc3FJtwzgh1Uv+Qelz9qeOH2bfWjS1s0s+y4rL

    // verify.go checks a .checkpoint of the Stonky transparency logs: that it is a checkpoint of the
    // log whose key is given (that log's origin, a size and a 32-byte root, signed with that key) and
    // that at least <n> of the witness keys given cosign it.
    //
    //	go run . <checkpoint> <n> <log key> <witness key>...
    package main

    import (
        "encoding/base64"
        "fmt"
        "os"
        "strconv"
        "strings"

        "github.com/transparency-dev/formats/log"
        "github.com/transparency-dev/formats/note"
        sumdb "golang.org/x/mod/sumdb/note"
    )

    func main() {
        if len(os.Args) < 4 {
            fail("usage: go run . <checkpoint> <n> <log key> <witness key>...")
        }
        cp, err := os.ReadFile(os.Args[1])
        check(err)
        witnesses := os.Args[4:]
        n, err := strconv.Atoi(os.Args[2])
        if err != nil || n < 0 || n > len(witnesses) {
            fail(fmt.Sprintf("<n> must be a number from 0 to %d, the witness keys given", len(witnesses)))
        }
        logKey, err := sumdb.NewVerifier(os.Args[3])
        check(err)
        // A cosignature does not sign the witness's name: one key given under two names, in either of
        // its encodings (0x01, 0x04), or the log's key given as a witness's, would count twice. Keys are
        // compared as their 32 bytes.
        given := map[string]string{ed25519Key(os.Args[3]): os.Args[3]}
        var keys []sumdb.Verifier
        for _, k := range witnesses {
            v, err := note.NewVerifierForCosignatureV1(k)
            check(err)
            if first, ok := given[ed25519Key(k)]; ok {
                fail("one key given twice: " + first + " and " + k)
            }
            given[ed25519Key(k)] = k
            keys = append(keys, v)
        }
        // This fails when the log's signature is missing, when the first signature line of one of the
        // keys given does not verify (a further line under the same name and key hash is neither
        // checked nor counted), and when the text is no checkpoint of the log's origin.
        c, _, opened, err := log.ParseCheckpoint(cp, logKey.Name(), logKey, keys...)
        check(err)
        if len(c.Hash) != 32 {
            fail(fmt.Sprintf("the root hash has %d bytes, not 32", len(c.Hash)))
        }
        cosigned := len(opened.Sigs) - 1 // all but the log's own signature
        if cosigned < n {
            fail(fmt.Sprintf("cosigned by %d of the witnesses given, not %d", cosigned, n))
        }
        fmt.Printf("OK: %s, %d entries, signed by the log, cosigned by %d of the %d witnesses given\n", c.Origin, c.Size, cosigned, len(witnesses))
    }

    // ed25519Key is the 32-byte Ed25519 key of a verifier key, <name>+<hash>+<base64 of its type and
    // key>, the type 0x01 (Ed25519) or 0x04 (Ed25519 cosignature/v1).
    func ed25519Key(vkey string) string {
        parts := strings.SplitN(vkey, "+", 3)
        raw, err := base64.StdEncoding.DecodeString(parts[len(parts)-1])
        if len(parts) != 3 || err != nil || len(raw) != 33 || (raw[0] != 0x01 && raw[0] != 0x04) {
            fail("not an Ed25519 verifier key: " + vkey)
        }
        return string(raw[1:])
    }

    func check(err error) {
        if err != nil {
            fail(err.Error())
        }
    }

    func fail(msg string) {
        fmt.Fprintln(os.Stderr, msg)
        os.Exit(1)
    }

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
