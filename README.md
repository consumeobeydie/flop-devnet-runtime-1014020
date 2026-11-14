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
