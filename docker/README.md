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
pulls, since the image only exists locally:

    -P ubuntu-26.04=localhost/shiranekowrt-act-runner:latest
    -P ubuntu-latest=localhost/shiranekowrt-act-runner:latest
    --pull=false

Run jobs with `act` or the VS Code act-visual-runner extension.
