# Running a FLOP Devnet Miner When the CLI Rejects Runtime 1014020

A field-tested workaround for setting up a FLOP Devnet GPU miner while the official CLI cannot talk to the current devnet runtime.

> **Status (14 November 2026):** working. A miner set up with this method took paid sessions within its first day.
> This is not an official FLOP Labs guide. The devnet gives no rewards and can be reset.

---

## TL;DR

- On 12–13 November 2026 the FLOP Devnet runtime is at spec version **1014020**.
- The newest published CLI image (**v0.1.321**, bundled SDK 0.5.2) only accepts runtimes up to **1014014**, so `model fetch` and `miner status` fail with `Unsupported runtime spec version 1014020`. See [flop-labs/network#238](https://github.com/flop-labs/network/issues/238).
- The miner **worker v0.1.336** runs fine against 1014020 in our test.
- The only blocking step is downloading the model. You can download the 10 model files directly from Hugging Face and let the **backend verify them against the on-chain model hash** on startup. If the files do not match, the backend refuses to start, so nothing unverified ever serves.

This guide builds on [UfukNode's FLOP miner setup guide](https://github.com/UfukNode/flop-miner-setup). Follow that guide, and replace only the steps described below.

---

## What needs the CLI, and what to do instead

| Step in UfukNode's guide | Needs a working CLI? | What we did |
|---|---|---|
| 9. Create the miner wallet | No (key commands work offline) | Worked as written. To reuse an existing account, see [Restoring an existing miner account](#restoring-an-existing-miner-account). |
| 10. Stake sponsor | Not tested on 1014020 | Our account was already registered from an earlier run, so we skipped this. |
| 11. Download and verify the model | **Yes, fails** | **Download from Hugging Face, let the backend verify.** See below. |
| 15. `miner status` | **Yes, fails** | Check the account on the explorer and use the worker's own `/session/info` check. |
| 16. Declare the GPU (optional) | Yes | Skipped. |
| 17. Calibration | Partly | Follow the worker log. In our run the worker read the existing calibration from the chain on its own. |
| 18–20. Fence guard, tunnel watch, speed patch | No | Worked as written. |

---

## The workaround: model download without the CLI

Run this on the server **instead of step 11**, after steps 1–10 of UfukNode's guide:

```bash
D=/opt/flop-miner/models/qwen2.5-0.5b-instruct
mkdir -p "$D"
for f in .gitattributes LICENSE README.md config.json generation_config.json merges.txt \
         model.safetensors tokenizer.json tokenizer_config.json vocab.json; do
  curl -fsSL -o "$D/$f" "https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct/resolve/main/$f" && echo "OK $f"
done
chmod -R a+rX /opt/flop-miner/models
ls -la "$D"
```

These are the same 10 files the CLI downloaded for us on 7 November 2026, when it still worked and verified them against the chain. `model.safetensors` should be **988,097,824 bytes**.

The `chmod` matters if your shell has a strict `umask` (for example `077` from the wallet step): the files are not secret, and the backend must be able to read them.

### Let the backend verify the files

Start the backend as in step 12:

```bash
cd /opt/flop-miner
compose up -d backend
compose logs -f backend
```

The **first line** of the log is the check that matters:

```
MODEL_DIR reproduces MODEL_HASH 1a52c1abe781db051e70ae357f72bb70161ca0c092df0b618f22a9292ba63306 (verity-v1)
```

If you see this line, the files match the registered model hash exactly. Wait for `Uvicorn running` and continue with step 13.

If the backend reports a mismatch instead, stop: the files on Hugging Face are not the registered checkpoint, and the miner must not run with them.

The worker checks the model a second time when it starts (`model-gate OK ... verified via /models/checkpoint`).

---

## Checking the miner without `miner status`

The worker answers this on the server, no CLI needed:

```bash
T=$(grep '^SESSION_AUTH_TOKEN=' /opt/flop-miner/miner.env | cut -d= -f2)
curl -s -H "Authorization: Bearer $T" http://127.0.0.1:8645/session/info | jq '{is_ready, audit_handoff_enabled}'
```

Both values must be `true`. With the fence guard from step 18 installed:

```bash
cd /opt/flop-miner
compose exec -T worker node --input-type=module < fence-guard/status.mjs | jq '{fenced, ready, minerAddress}'
```

For the on-chain view (stake, balance, payments), open your account on the explorer:
`https://explorer.devnet.flop.network/account/YOUR_MINER_ADDRESS`

> When you share logs for help, remove your tunnel address (`*.trycloudflare.com`), your server IP and SSH port first. `SESSION_AUTH_TOKEN`, `INFERENCE_API_KEY` and the files in `signer`, `session-keys/` and `audit-key/` must never leave the server.

---

## Restoring an existing miner account

If your account is already registered (for example from a server you destroyed), do not create a new wallet and do not run the sponsor again: the sponsor registers each account only once. Write your 24 words back into the `signer` file without them appearing on screen or in your shell history:

```bash
cd /opt/flop-miner
umask 077
read -r -s -p 'Enter the 24 words, then Enter: ' M && echo \
  && printf '%s\n' "$M" > /opt/flop-miner/signer && unset M \
  && chmod 600 signer && wc -w < signer
```

The last line must print `24`. Then check that the words give your address, before the worker ever uses them:

```bash
export FLOP_KEY_PASSWORD=$(openssl rand -hex 16)
MINER_SECRET="$(cat /opt/flop-miner/signer)" docker run --rm -e MINER_SECRET -e FLOP_KEY_PASSWORD \
  -v /opt/flop-miner:/work "$FLOP_CLI" keys import --type sr25519 --secret-env MINER_SECRET \
  --out /work/verify.json --password-env FLOP_KEY_PASSWORD
jq -r .address /opt/flop-miner/verify.json
rm -f /opt/flop-miner/verify.json
unset FLOP_KEY_PASSWORD
```

`keys import` does not contact the chain, so it works with CLI v0.1.321. Create the session key and the evidence key as in step 9, and start the worker as in step 14. The `Signer:` line in the worker log must show your address.

> **Security:** run these commands only on a server that nobody else can log in to. While the check runs, the words are passed to a short-lived container through an environment variable, and the `signer` file itself holds them in plain text, as in the original guide. Never paste the words, the `signer` file or any `.env` content into a chat, an issue or a screenshot.

---

## Results

| | |
|---|---|
| GPU | 1x RTX 5060 Ti 16 GB, Vast.ai Ubuntu 22.04 VM, $0.209/hour |
| Driver | 580.95.05, CUDA 13.0 |
| Versions | worker v0.1.336 (with UfukNode's speed patch), backend v0.1.239, CLI v0.1.321 |
| Devnet runtime | 1014020 |
| Model check | `MODEL_DIR reproduces MODEL_HASH 1a52c1ab…a63306` with files from Hugging Face |
| Calibration | Existing 130 GFLOP/s calibration read from the chain; `is_ready: true` right after start |
| First ~19 hours | 41 sessions opened, **7 paid**, 4 session errors |
| Work paid | 1,080 G_n in total (largest session 530 G_n) |
| Earned | ~3.22 test FLOP (about 0.003 test FLOP per G_n, less 1 %) |

Two notes on these numbers:

- **Most opened sessions are FLOP's near-zero test sessions** (`maxUnackedGn=0` in the log). UfukNode's guide reports the same on every miner, so they need no fix.
- **Fees are larger than earnings for small sessions.** UfukNode's guide measured about 2.63 test FLOP in fees per paid single-turn session on worker v0.1.336, so a session needs roughly 885 G_n to cover its fees. All 7 of our paid sessions were below that. Test FLOP has no value, so this matters only for the fee balance the sponsor gives (250 test FLOP); the faucet at `https://faucet.flop.network` adds 25 per day.

### The 4 session errors

```
[session] error: AggregateError: TOPLOC background publication exhausted; evidence remains unpublished
```

The worker could not publish the evidence for these sessions, most likely during drops of the public RPC. In our run the worker stayed `is_ready: true` and `fenced: false` afterwards, and kept taking and settling sessions. If your worker stays `is_ready: false` after an RPC outage, see *The worker stays is_ready: false after an RPC outage* in UfukNode's guide.

---

## Vast.ai notes

- If a new instance fails with `toomanyrequests: You have reached your unauthenticated pull rate limit`, the **host** hit Docker Hub's limit while pulling the VM image. It is not your setup. Destroy it and rent a machine on a **different host** (the `host:` number in the listing).
- Check that the GPU really has **16 GB**: the RTX 5060 Ti also exists with 8 GB, and the plain RTX 5060 is 8 GB. Filter with *Per GPU RAM ≥ 16*.
- The Ubuntu 22.04 VM template sets a 130 GB disk by default, which is plenty.

---

## When to stop using this workaround

As soon as FLOP publishes a CLI that accepts the current devnet runtime (watch [flop-labs/network#238](https://github.com/flop-labs/network/issues/238) and `https://flop.network/releases.json`), go back to the official `model fetch` in step 11. The backend check above protects you either way, but the official path verifies the files before they are even written to disk.

---

## Credits

- [UfukNode/flop-miner-setup](https://github.com/UfukNode/flop-miner-setup) by [@UfukDegen](https://x.com/UfukDegen): the full setup, the fence guard, the tunnel watch and the speed patch used here.
- [flop-labs/network#238](https://github.com/flop-labs/network/issues/238): the original report of the CLI / runtime 1014020 mismatch.

---

**Author:** [consumeobeydie](https://github.com/consumeobeydie)
**Technocore DID:** `did:key:z6Mkp2F9qv7Z7ZPbch3uhtGS5wBsK9n1Bvm3GpCU8xzkKhFw`
