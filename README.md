# obscura-stealth-builder

CI that rebuilds [`h4ckf0r0day/obscura`](https://github.com/h4ckf0r0day/obscura)
with the **`stealth`** cargo feature turned on, and pushes the resulting
multi-arch image to **GHCR** under your own account.

The upstream Docker image ships `--features render` only — the maintainer
deliberately leaves `stealth` out (see their README note). This repo adds it
back without forking: every run checks out upstream's source at its latest
release tag, patches the one build line, and builds. There is nothing to keep
in sync — you always track upstream's newest release automatically.

## What it produces

```
ghcr.io/<your-account>/obscura:<version>   # e.g. :0.2.2
ghcr.io/<your-account>/obscura:latest
```

Built for `linux/amd64` and `linux/arm64`, same as upstream, plus stealth.

## Setup (one time)

1. Create a **new empty GitHub repo** (any name, e.g. `obscura-stealth-builder`)
   and push these two files:
   ```
   .github/workflows/build.yml
   README.md
   ```
2. That's it for credentials — the workflow authenticates to GHCR with the
   built-in `GITHUB_TOKEN` (`packages: write`). No Docker Hub secrets needed.
3. Open the **Actions** tab. If prompted, enable workflows for the repo.

## First build

- Actions tab → **Build Obscura (stealth)** → **Run workflow**.
- Leave inputs blank to build upstream's latest release, or type a specific
  `tag` (e.g. `v0.2.2`). Tick **force** to rebuild a version that already exists.

After the first successful push, make the package usable:

- Your profile → **Packages** → `obscura` → **Package settings**.
- Set visibility to **Public** if you want to `docker pull` without logging in
  (GHCR packages start private).

## Ongoing

The `schedule:` cron runs daily at 06:00 UTC. Each run:

1. resolves upstream's latest release tag,
2. **skips** if that version is already in your GHCR (no wasted build),
3. otherwise checks out that tag, enables stealth, builds, and pushes.

So a new upstream release turns into a stealth image within ~24h, untouched.

## Using the image

```bash
docker run -d --name obscura -p 127.0.0.1:9222:9222 \
  ghcr.io/<your-account>/obscura:latest
```

> The CDP control plane has **no authentication** — anything that can reach
> port 9222 can drive the browser. Keep it bound to loopback
> (`-p 127.0.0.1:9222:9222`) or behind an auth proxy. This is inherited from
> upstream, unchanged.

## How the stealth patch works

Upstream `Dockerfile` builds with:

```
cargo build --release --features render --bin obscura --bin obscura-worker
```

The workflow does `sed 's/--features render /--features render,stealth /g'`,
producing `--features render,stealth`. The `stealth` feature is defined in
`crates/obscura-cli/Cargo.toml`:

```
stealth = ["obscura-browser/stealth", "obscura-net/stealth", "obscura-mcp/stealth"]
```

If a future upstream release restructures that build line, the workflow fails
loudly (the "Enable the stealth feature" step asserts the patch took effect)
rather than silently shipping a non-stealth image.

## License

The built artifact is Obscura, which is Apache-2.0. This builder only
reconfigures its compile-time features; all copyright remains upstream's.
