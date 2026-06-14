# Contributing

Thanks for your interest in improving this role! This guide covers how to set
up a development environment, run the linters and formatters, and execute the
test suite locally so your changes match what CI expects.

## Ways to contribute

- **Report bugs** or request features by opening an [issue](https://github.com/sebdanielsson/ansible-role-xcaddy/issues).
- **Submit changes** via a pull request from a topic branch (see [Pull requests](#pull-requests)).

## Prerequisites

- **Python** 3.14 (the pinned version lives in [`.python-version`](.python-version))
- **[uv](https://docs.astral.sh/uv/)** for managing the Python environment and dependencies
- **Docker** — required by Molecule to spin up test containers
- **[Go](https://go.dev/)** (optional) — only needed to run `yamlfmt` the same way CI does

## Development environment

### 1. Clone the repository

```sh
git clone https://github.com/sebdanielsson/ansible-role-xcaddy.git
cd ansible-role-xcaddy
```

### 2. Create and activate a virtual environment

`uv` reads `.python-version` and provisions the correct interpreter automatically.

```sh
uv venv
source .venv/bin/activate
```

### 3. Install Python dependencies

```sh
uv pip install -r requirements.txt
```

This installs `ansible-core`, `molecule`, the Molecule Docker plugin, and the
`docker` SDK — all pinned in [`requirements.txt`](requirements.txt).

### 4. Install the required Ansible collections

```sh
ansible-galaxy collection install -r requirements.yml
```

### 5. Verify the installation

```sh
ansible --version
ansible-lint --version
molecule --version
```

## Linting

CI enforces two checks: `ansible-lint` and a `yamlfmt` formatting check. Run both
before pushing.

### ansible-lint

The role targets the **production** profile with zero findings.

```sh
ansible-lint
```

Configuration lives in [`.ansible-lint`](.ansible-lint). `ansible-lint` also runs
`yamllint` internally using the rules in [`.yamllint.yaml`](.yamllint.yaml), so YAML
style is covered as part of this check.

You can optionally run `yamllint` on its own for a quick YAML-only pass (not a
separate CI step):

```sh
yamllint .
```

## Formatting

YAML is formatted with [`yamlfmt`](https://github.com/google/yamlfmt), configured
in [`.yamlfmt.yaml`](.yamlfmt.yaml). CI runs it in lint-only mode and fails if any
file is not formatted.

Check formatting (matches CI):

```sh
go run github.com/google/yamlfmt/cmd/yamlfmt@main -conf .yamlfmt.yaml -lint
```

Apply formatting in place:

```sh
go run github.com/google/yamlfmt/cmd/yamlfmt@main -conf .yamlfmt.yaml
```

> If you have `yamlfmt` installed on your `PATH`, you can call it directly
> instead of via `go run`.

Markdown is linted against [`.markdownlint.json`](.markdownlint.json) if you use
a markdownlint-compatible tool/editor; it is not currently enforced in CI.

## Testing

Tests use [Molecule](https://ansible.readthedocs.io/projects/molecule/) with the
Docker driver. The default scenario lives in [`molecule/default/`](molecule/default/).

Run the full test sequence (create → converge → verify → destroy):

```sh
molecule test
```

While iterating, it is faster to converge and verify against a persistent
container instead of recreating it each time:

```sh
molecule converge   # apply the role
molecule verify     # run the assertions in verify.yml
molecule login      # shell into the test container for debugging
molecule destroy    # tear the container down when done
```

### Testing against different distributions

CI runs the suite across Debian, Ubuntu, Fedora, and RHEL (UBI). Locally you can
target a specific image with the same environment variables CI uses (defaults to
`ubuntu:latest`):

```sh
MOLECULE_DOCKER_IMAGE=debian MOLECULE_DOCKER_TAG=13 molecule test
MOLECULE_DOCKER_IMAGE=fedora MOLECULE_DOCKER_TAG=43 molecule test
MOLECULE_DOCKER_IMAGE=redhat/ubi10 MOLECULE_DOCKER_TAG=10.1 molecule test
```

### What the tests cover

- **`converge.yml`** drives the role through staged scenarios that assert build
  idempotency (initial build → no-op re-run → rebuild on version change).
- **`verify.yml`** asserts the resulting state: binary, version, modules, config,
  service status, and HTTP response.

When adding a feature, extend `converge.yml` and/or `verify.yml` so the behaviour
is covered.

## Pull requests

1. Create a topic branch off `main` (e.g. `feat/...`, `fix/...`, `docs/...`).
2. Make your change, keeping it focused and idempotent.
3. Run the linters, formatter, and `molecule test` locally.
4. Document new variables in [`README.md`](README.md) and the `defaults/main.yml`
   comments.
5. Open a PR against `main`. CI must pass before merge.

## Releases

Releases are automated: pushing a Git tag triggers the
[`Release` workflow](.github/workflows/publish.yml), which imports the role to
[Ansible Galaxy](https://galaxy.ansible.com/). Maintainers handle tagging.
