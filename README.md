# Sandbox Git Mirror

A pinned **mirror** sandbox that checks out a repo, refreshes it nightly, and serves a
compressed archive (including `.git`) over an **internal-only** HTTPS endpoint. A
**client** sandbox then seeds its checkout from that archive instead of cloning from
GitHub on every creation.

Why: a direct `git clone` egresses through the AWS NAT Gateway on every sandbox. Here the
bulk transfer stays in-cluster over the internal endpoint, so only a tiny delta `git pull`
ever touches the network.

## How it works

- **`mirror-sandbox.yaml`** — clones the repo, a `post-checkout` hook tars just the
  `.git` dir to `~/mirror-out/hello.tar.gz`, a daemon serves it on port 8000, and a
  nightly job (`0 22 * * *`) does `git pull` + repack. Exposed via an `INTERNAL`
  endpoint (auth proxy disabled).
- **`client-sandbox.yaml`** — a repo-less `path` checkout whose `post-checkout` hook
  streams the tarball straight into the checkout (`curl -k ... | tar -zx`), runs
  `git checkout -f master` to restore the working tree, then `git pull` for the delta.
  `origin` comes from the archive's `.git`, so no Git remote is declared.

## Setup (3 steps)

1. **Create and pin the mirror:**
   ```bash
   cs sandbox create mirror --from def:mirror-sandbox.yaml --wait
   cs sandbox pin mirror
   ```

2. **Get the mirror's internal host** and put it in `client-sandbox.yaml`
   (replace `__MIRROR_HOST__`):
   ```bash
   cs sandbox show mirror   # copy the *.sandboxes.internal URL host
   ```

3. **Create the client:**
   ```bash
   cs sandbox create client --from def:client-sandbox.yaml --wait
   ```

Verify: `cs exec -W client/client -u 1000 -- bash -lc 'cd ~/hello && git status && git remote -v'`
— the tree is clean, current, and `origin` points at the source repo.

## Notes

- `-k` is required because the internal endpoint uses an internal-CA cert.
- To mirror a different repo, change the `repo.git` URL in `mirror-sandbox.yaml`.
- These definitions don't define a build step — that's left to you. The example `hello`
  repo ships its own `.sandbox/build` (`go build`), so it expects Go in the workspace;
  point this at your own repo and define whatever build you need after checkout.
- Tarring only `.git` keeps the archive small; the client rebuilds the working tree
  from it with `git checkout`.
- Instead of the nightly job, you can run the mirror workspace in
  [Auto mode](https://docs.sandboxes.cloud/features/workspace-automation.html#auto-mode)
  (e.g. `cs sandbox create mirror --from def:mirror-sandbox.yaml -A '*'`): the
  `post-checkout` hook then re-runs on every change Auto mode pulls, so you can drop the
  `jobs` block. Auto mode is set at create time, not in the definition file.
