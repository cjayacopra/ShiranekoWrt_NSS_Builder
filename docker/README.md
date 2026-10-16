# Local docker images

Everything needed to build the images lives in this directory - nothing
references paths outside the repository.

## builder-base (`docker/builder-base/Containerfile`)

The OpenWrt buildroot host toolchain image (debian slim + the full compile
toolchain, non-root `buildbot` user). Tag: `localhost/shiranekowrt-builder:base`.

    docker build -t localhost/shiranekowrt-builder:base -f docker/builder-base/Containerfile docker/builder-base/

## act-runner (`docker/act-runner/Containerfile`)

The local GitHub Actions runner on top of the base image: adds node (for JS
actions), gh, curl, jq, passwordless sudo. Tag: `localhost/shiranekowrt-act-runner:latest`.

    docker build -t localhost/shiranekowrt-act-runner:latest -f docker/act-runner/Containerfile docker/act-runner/

## Local runs

The repo-level `.actrc` (git-ignored, machine-specific) maps the workflows'
`runs-on: ubuntu-26.04` label to the act-runner image and disables image
pulls, since the image only exists locally. It also enables act's new action
cache with offline mode (actions are fetched from github once, then served
from `~/.cache/act` - the old cache re-cloned every run and repeatedly died
with `unexpected EOF` on this machine's flaky github link), and mounts the
repo's `.local-mirrors/` directory read-only at `/mnt/git-mirrors`:

    -P ubuntu-26.04=localhost/shiranekowrt-act-runner:latest
    -P ubuntu-latest=localhost/shiranekowrt-act-runner:latest
    --pull=false
    --use-new-action-cache
    --action-offline-mode
    --container-options=-v <repo-abs-path>/.local-mirrors:/mnt/git-mirrors:ro

## Upstream source mirror (`.local-mirrors/`)

The *Build aw1000* job clones the multi-GB `openwrt-nss-edma` upstream with
full history; over a flaky link that clone routinely dies mid-transfer
(`fatal: early EOF`). The act-runner image contains a `git url.insteadOf`
rewrite (baked at image build, see its Containerfile) pointing that clone at
`/mnt/git-mirrors/openwrt-nss-edma` - the local mirror mounted from
`.local-mirrors/`. Create it once per machine:

    mkdir -p .local-mirrors
    git init --bare -q .local-mirrors/openwrt-nss-edma
    git -C .local-mirrors/openwrt-nss-edma remote add origin \
      https://github.com/JuliusBairaktaris/openwrt-nss-edma.git
    git -C .local-mirrors/openwrt-nss-edma fetch origin '+refs/*:refs/*'

The fetch is resumable - just re-run it until it completes. GitHub runs are
unaffected: the rewrite only exists inside the local act-runner image.

Run jobs with `act` or the VS Code act-visual-runner extension.
