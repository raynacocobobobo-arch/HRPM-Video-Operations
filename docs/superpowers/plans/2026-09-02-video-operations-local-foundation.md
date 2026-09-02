# Video Operations Local Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a reproducible, pinned, localhost-only OpenProject 17.7.2 foundation on the user's Apple Silicon Mac without modifying the existing HRPM.

**Architecture:** A new Git repository records exact upstream versions and uses the official OpenProject Docker Compose repository as a pinned submodule. Colima supplies an isolated Docker runtime; OpenProject, PostgreSQL, cache, worker, cron, and proxy run from upstream definitions with only local environment configuration.

**Tech Stack:** POSIX shell, Python 3.11 `unittest`, Git submodules, Colima 0.10.3, Docker CLI/Compose, OpenProject 17.7.2, PostgreSQL 17.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Global Constraints

- Repository root is the current Git clone, resolved with `git rev-parse --show-toplevel`.
- Stop before installation unless `/System/Volumes/Data` has at least 30 GiB free.
- Do not delete or move user data as part of preflight.
- Bind the only browser-facing port to `127.0.0.1:8080`.
- Pin OpenProject `17.7.2`, Compose commit `1cf58dc832fb803ee44fa7632449ce8f8f2b928f`, prototype-plugin reference commit `48bd2359632d22c72836798dbea566f9544050fd`, Timefold quickstarts reference commit `b9abb3bcd417d51cbd972a69744ba9fc81173b7f`, PostgreSQL `17`, and Colima `0.10.3`.
- Do not install ERPNext, Plane, Kimai, Leantime, or the Timefold service in this package.
- Do not import HRPM data in this package.
- Commit `.env.example`; never commit `.env`, passwords, API tokens, database data, or attachments.

---

## Planned File Map

```text
video-operations/
├── .gitignore                         # secrets, runtime data, build output
├── README.md                          # local operator entrypoint
├── UPSTREAMS.md                       # source, license, version, commit inventory
├── versions.env                       # machine-readable version lock
├── Makefile                           # stable operator commands
├── compose/
│   ├── .env.example                   # localhost-only OpenProject configuration
│   └── docker-compose.override.yml    # pass Chinese locale into app services
├── ops/
│   ├── preflight.sh                   # disk/architecture prerequisite gate
│   ├── bootstrap-runtime.sh           # install/start Colima profile
│   ├── init-env.sh                    # create untracked secrets
│   ├── compose.sh                     # one canonical pinned Compose wrapper
│   ├── up.sh                          # start pinned upstream Compose stack
│   ├── down.sh                        # stop stack without deleting volumes
│   ├── smoke.sh                       # local health and bind-address checks
│   └── tests/
│       ├── test_preflight.py
│       ├── test_versions.py
│       └── test_local_compose.py
└── vendor/
    └── openproject-docker-compose/    # pinned Git submodule
```

### Task 1: Repository bootstrap and disk safety gate

**Files:**
- Create: `.gitignore`
- Create: `README.md`
- Create: `ops/preflight.sh`
- Test: `ops/tests/test_preflight.py`

**Interfaces:**
- Consumes: no previous package.
- Produces: `ops/preflight.sh [--min-free-gib N]`, exit `0` when safe and exit `2` when disk space is insufficient.
- Produces: one JSON line with keys `architecture`, `free_gib`, `minimum_gib`, and `ready`.

- [ ] **Step 1: Initialize the isolated repository**

Run:

```bash
cd "$(git rev-parse --show-toplevel)"
test -f HANDOFF.md
test "$(git branch --show-current)" = "main"
mkdir -p ops/tests compose vendor
```

Expected: the repository is the private `HRPM-Video-Operations` clone, `git status --short` is empty, and `git branch --show-current` prints `main`.

- [ ] **Step 2: Write the failing preflight tests**

Create `ops/tests/test_preflight.py`:

```python
import json
import os
import pathlib
import subprocess
import unittest


ROOT = pathlib.Path(__file__).resolve().parents[2]
SCRIPT = ROOT / "ops" / "preflight.sh"


class PreflightTest(unittest.TestCase):
    def run_preflight(self, free_kib: int):
        env = os.environ.copy()
        env["VIDEO_OPS_FREE_KIB_OVERRIDE"] = str(free_kib)
        return subprocess.run(
            [str(SCRIPT), "--min-free-gib", "30"],
            cwd=ROOT,
            env=env,
            text=True,
            capture_output=True,
        )

    def test_rejects_less_than_30_gib(self):
        result = self.run_preflight(29 * 1024 * 1024)
        self.assertEqual(result.returncode, 2)
        self.assertFalse(json.loads(result.stdout)["ready"])

    def test_accepts_30_gib(self):
        result = self.run_preflight(30 * 1024 * 1024)
        self.assertEqual(result.returncode, 0)
        self.assertTrue(json.loads(result.stdout)["ready"])

    def test_reports_arm64_on_this_mac(self):
        result = self.run_preflight(30 * 1024 * 1024)
        self.assertEqual(json.loads(result.stdout)["architecture"], "arm64")


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 3: Run the test to verify it fails**

Run:

```bash
python3 -m unittest ops.tests.test_preflight -v
```

Expected: ERROR with `No such file or directory: .../ops/preflight.sh`.

- [ ] **Step 4: Implement the minimal preflight gate**

Create `ops/preflight.sh`:

```sh
#!/bin/sh
set -eu

minimum_gib=30
if [ "${1:-}" = "--min-free-gib" ]; then
  minimum_gib="${2:?missing minimum GiB}"
fi

architecture="$(uname -m)"
if [ -n "${VIDEO_OPS_FREE_KIB_OVERRIDE:-}" ]; then
  free_kib="$VIDEO_OPS_FREE_KIB_OVERRIDE"
else
  free_kib="$(df -Pk /System/Volumes/Data | awk 'NR==2 {print $4}')"
fi

minimum_kib=$((minimum_gib * 1024 * 1024))
free_gib=$((free_kib / 1024 / 1024))
ready=false
exit_code=2

if [ "$architecture" = "arm64" ] && [ "$free_kib" -ge "$minimum_kib" ]; then
  ready=true
  exit_code=0
fi

printf '{"architecture":"%s","free_gib":%s,"minimum_gib":%s,"ready":%s}\n' \
  "$architecture" "$free_gib" "$minimum_gib" "$ready"
exit "$exit_code"
```

Run:

```bash
chmod +x ops/preflight.sh
```

- [ ] **Step 5: Add repository ignores and operator README**

Create `.gitignore`:

```gitignore
.DS_Store
.env
compose/.env
runtime/
backups/
restore-drills/
*.log
target/
node_modules/
```

Create `README.md`:

```markdown
# Video Operations

Local-only OpenProject + Video Operations + Timefold deployment.

## Safety gate

Run `./ops/preflight.sh --min-free-gib 30`. Do not install or start containers
until it exits 0. The script never deletes or moves user files.
```

- [ ] **Step 6: Run the tests and verify they pass**

Run:

```bash
python3 -m unittest ops.tests.test_preflight -v
```

Expected: 3 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add .gitignore README.md ops/preflight.sh ops/tests/test_preflight.py
git commit -m "chore: add local disk preflight gate"
```

### Task 2: Pin upstream repositories and application versions

**Files:**
- Create: `versions.env`
- Create: `UPSTREAMS.md`
- Create: `.gitmodules` via `git submodule add`
- Test: `ops/tests/test_versions.py`

**Interfaces:**
- Consumes: repository created in Task 1.
- Produces: shell-compatible exact version variables.
- Produces: `vendor/openproject-docker-compose` fixed at commit `1cf58dc832fb803ee44fa7632449ce8f8f2b928f`.

- [ ] **Step 1: Write the failing version-lock tests**

Create `ops/tests/test_versions.py`:

```python
import pathlib
import subprocess
import unittest


ROOT = pathlib.Path(__file__).resolve().parents[2]


class VersionLockTest(unittest.TestCase):
    def test_exact_versions_are_recorded(self):
        text = (ROOT / "versions.env").read_text()
        self.assertIn("OPENPROJECT_VERSION=17.7.2", text)
        self.assertIn("POSTGRES_VERSION=17", text)
        self.assertIn("COLIMA_VERSION=0.10.3", text)
        self.assertIn("TIMEFOLD_VERSION=2.6.0", text)
        self.assertIn("QUARKUS_VERSION=3.38.3", text)
        self.assertIn(
            "OPENPROJECT_PROTO_PLUGIN_COMMIT=48bd2359632d22c72836798dbea566f9544050fd",
            text,
        )
        self.assertIn(
            "TIMEFOLD_QUICKSTARTS_COMMIT=b9abb3bcd417d51cbd972a69744ba9fc81173b7f",
            text,
        )

    def test_compose_submodule_is_pinned(self):
        actual = subprocess.check_output(
            ["git", "-C", "vendor/openproject-docker-compose", "rev-parse", "HEAD"],
            cwd=ROOT,
            text=True,
        ).strip()
        self.assertEqual(actual, "1cf58dc832fb803ee44fa7632449ce8f8f2b928f")


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the tests to verify they fail**

Run:

```bash
python3 -m unittest ops.tests.test_versions -v
```

Expected: ERROR because `versions.env` and the submodule do not exist.

- [ ] **Step 3: Add and pin the official Compose repository**

Run:

```bash
git submodule add -b stable/17 https://github.com/opf/openproject-docker-compose.git vendor/openproject-docker-compose
git -C vendor/openproject-docker-compose checkout 1cf58dc832fb803ee44fa7632449ce8f8f2b928f
```

Expected: `git submodule status` begins with `1cf58dc832fb...`.

- [ ] **Step 4: Create the machine-readable version lock**

Create `versions.env`:

```dotenv
OPENPROJECT_VERSION=17.7.2
OPENPROJECT_COMPOSE_COMMIT=1cf58dc832fb803ee44fa7632449ce8f8f2b928f
POSTGRES_VERSION=17
COLIMA_VERSION=0.10.3
TIMEFOLD_VERSION=2.6.0
QUARKUS_VERSION=3.38.3
JAVA_VERSION=21
OPENPROJECT_PROTO_PLUGIN_COMMIT=48bd2359632d22c72836798dbea566f9544050fd
TIMEFOLD_QUICKSTARTS_COMMIT=b9abb3bcd417d51cbd972a69744ba9fc81173b7f
```

Create `UPSTREAMS.md`:

```markdown
# Upstream inventory

| Component | Source | Pin | License |
|---|---|---|---|
| OpenProject | https://github.com/opf/openproject | v17.7.2 | GPL-3.0 |
| OpenProject Compose | https://github.com/opf/openproject-docker-compose | 1cf58dc832fb803ee44fa7632449ce8f8f2b928f | GPL-3.0 |
| OpenProject prototype plugin | https://github.com/opf/openproject-proto_plugin | 48bd2359632d22c72836798dbea566f9544050fd | GPL-3.0 |
| Timefold Solver | https://github.com/TimefoldAI/timefold-solver | v2.6.0 | Apache-2.0 |
| Timefold quickstarts | https://github.com/TimefoldAI/timefold-quickstarts | b9abb3bcd417d51cbd972a69744ba9fc81173b7f | Apache-2.0 |
| Colima | https://github.com/abiosoft/colima | v0.10.3 | MIT |
```

The prototype plugin and employee-scheduling quickstart are read-only pattern references, not vendored runtime dependencies. Copy no pattern unless the inspected checkout matches the recorded 40-character commit.

- [ ] **Step 5: Run the tests and verify they pass**

Run:

```bash
python3 -m unittest ops.tests.test_versions -v
```

Expected: 2 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add .gitmodules vendor/openproject-docker-compose versions.env UPSTREAMS.md ops/tests/test_versions.py
git commit -m "chore: pin OpenProject and runtime upstreams"
```

### Task 3: Install and configure the isolated Colima runtime

**Files:**
- Create: `ops/bootstrap-runtime.sh`
- Test: `ops/tests/test_local_compose.py`

**Interfaces:**
- Consumes: `ops/preflight.sh` and `versions.env`.
- Produces: Colima profile `video-operations` with 4 CPUs, 8 GiB RAM, 30 GiB sparse disk, Docker runtime, and no network-address exposure.

- [ ] **Step 1: Add the failing runtime policy test**

Create `ops/tests/test_local_compose.py`:

```python
import pathlib
import unittest


ROOT = pathlib.Path(__file__).resolve().parents[2]


class LocalRuntimePolicyTest(unittest.TestCase):
    def test_colima_profile_is_isolated(self):
        text = (ROOT / "ops" / "bootstrap-runtime.sh").read_text()
        self.assertIn("--profile video-operations", text)
        self.assertIn("--cpus 4", text)
        self.assertIn("--memory 8", text)
        self.assertIn("--disk 30", text)
        self.assertNotIn("--network-address", text)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
python3 -m unittest ops.tests.test_local_compose.LocalRuntimePolicyTest.test_colima_profile_is_isolated -v
```

Expected: ERROR because `ops/bootstrap-runtime.sh` does not exist.

- [ ] **Step 3: Implement the runtime bootstrap script**

Create `ops/bootstrap-runtime.sh`:

```sh
#!/bin/sh
set -eu

repo_root="$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)"
"$repo_root/ops/preflight.sh" --min-free-gib 30

if ! command -v brew >/dev/null 2>&1; then
  printf '%s\n' 'Homebrew is required and was not found.' >&2
  exit 3
fi

brew install colima docker docker-compose

installed_version="$(colima version | awk 'NR==1 {sub(/^colima version /, ""); print $1}')"
required_version="$(awk -F= '$1=="COLIMA_VERSION" {print $2}' "$repo_root/versions.env")"
if [ "$installed_version" != "$required_version" ]; then
  printf 'Expected Colima %s, found %s\n' "$required_version" "$installed_version" >&2
  exit 4
fi

colima start \
  --profile video-operations \
  --runtime docker \
  --vm-type vz \
  --cpus 4 \
  --memory 8 \
  --disk 30

docker context use colima-video-operations
docker version
docker compose version
```

Run:

```bash
chmod +x ops/bootstrap-runtime.sh
```

- [ ] **Step 4: Run the policy test**

Run:

```bash
python3 -m unittest ops.tests.test_local_compose.LocalRuntimePolicyTest.test_colima_profile_is_isolated -v
```

Expected: PASS.

- [ ] **Step 5: Run the real preflight and stop if it fails**

Run:

```bash
./ops/preflight.sh --min-free-gib 30
```

Expected before user cleanup: exit 2 with `"ready":false`. Do not run the next step until the user has freed disk space and this command exits 0.

- [ ] **Step 6: Bootstrap and verify the runtime after approval**

Run:

```bash
./ops/bootstrap-runtime.sh
colima status --profile video-operations
docker context show
```

Expected: Colima reports `Running` and Docker context is `colima-video-operations`.

- [ ] **Step 7: Commit**

```bash
git add ops/bootstrap-runtime.sh ops/tests/test_local_compose.py
git commit -m "chore: add isolated Colima runtime bootstrap"
```

### Task 4: Start pinned vanilla OpenProject on localhost

**Files:**
- Create: `compose/.env.example`
- Create: `compose/docker-compose.override.yml`
- Create: `ops/init-env.sh`
- Create: `ops/compose.sh`
- Create: `ops/up.sh`
- Create: `ops/down.sh`
- Create: `ops/smoke.sh`
- Create: `Makefile`
- Modify: `ops/tests/test_local_compose.py`

**Interfaces:**
- Consumes: Docker context `colima-video-operations` and pinned Compose submodule.
- Produces: `./ops/compose.sh ARGS`, `make up`, `make down`, and `make smoke`.
- Produces: one local browser URL, `http://localhost:8080`; the listener remains bound to `127.0.0.1:8080`.

- [ ] **Step 1: Extend the failing localhost configuration tests**

Append to `LocalRuntimePolicyTest` in `ops/tests/test_local_compose.py`:

```python
    def test_example_environment_is_loopback_only(self):
        text = (ROOT / "compose" / ".env.example").read_text()
        self.assertIn("TAG=17.7.2-slim", text)
        self.assertIn("POSTGRES_VERSION=17", text)
        self.assertIn("PORT=127.0.0.1:8080", text)
        self.assertIn("OPENPROJECT_HOST__NAME=localhost", text)
        self.assertIn("OPENPROJECT_HTTPS=false", text)
        self.assertNotIn("0.0.0.0", text)

    def test_compose_wrapper_includes_local_override(self):
        text = (ROOT / "ops" / "compose.sh").read_text()
        self.assertIn("vendor/openproject-docker-compose/docker-compose.yml", text)
        self.assertIn("compose/docker-compose.override.yml", text)

    def test_down_does_not_delete_volumes(self):
        text = (ROOT / "ops" / "down.sh").read_text()
        self.assertIn(" down", text)
        self.assertNotIn("--volumes", text)
        self.assertNotIn(" -v", text)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run:

```bash
python3 -m unittest ops.tests.test_local_compose -v
```

Expected: ERROR for missing `.env.example` and `down.sh`.

- [ ] **Step 3: Create the environment template and secret initializer**

Create `compose/.env.example`:

```dotenv
TAG=17.7.2-slim
POSTGRES_VERSION=17
PORT=127.0.0.1:8080
OPENPROJECT_HOST__NAME=localhost
OPENPROJECT_URL=http://web:8080
OPENPROJECT_HTTPS=false
OPENPROJECT_HSTS=false
OPENPROJECT_DEFAULT__LANGUAGE=zh-CN
PGDATA=pgdata
OPDATA=opdata
SECRET_KEY_BASE=GENERATE_ME
POSTGRES_PASSWORD=GENERATE_ME
DATABASE_URL=postgres://postgres:GENERATE_ME@db/openproject?pool=20&encoding=unicode&reconnect=true
COLLABORATIVE_SERVER_SECRET=GENERATE_ME
```

Create `compose/docker-compose.override.yml` so the locale variable—unlike the variables already referenced by upstream Compose—is explicitly passed to all app processes:

```yaml
x-video-operations-environment: &video_operations_environment
  OPENPROJECT_DEFAULT__LANGUAGE: "${OPENPROJECT_DEFAULT__LANGUAGE:-zh-CN}"

services:
  web:
    environment: *video_operations_environment
  worker:
    environment: *video_operations_environment
  cron:
    environment: *video_operations_environment
  seeder:
    environment: *video_operations_environment
```

Create `ops/init-env.sh`:

```sh
#!/bin/sh
set -eu

repo_root="$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)"
target="$repo_root/compose/.env"
if [ -e "$target" ]; then
  printf '%s\n' 'compose/.env already exists; refusing to overwrite.' >&2
  exit 2
fi

secret_key="$(openssl rand -hex 64)"
postgres_password="$(openssl rand -hex 32)"
collab_secret="$(openssl rand -hex 32)"

sed \
  -e "s/SECRET_KEY_BASE=GENERATE_ME/SECRET_KEY_BASE=$secret_key/" \
  -e "s/POSTGRES_PASSWORD=GENERATE_ME/POSTGRES_PASSWORD=$postgres_password/" \
  -e "s#DATABASE_URL=postgres://postgres:GENERATE_ME@#DATABASE_URL=postgres://postgres:$postgres_password@#" \
  -e "s/COLLABORATIVE_SERVER_SECRET=GENERATE_ME/COLLABORATIVE_SERVER_SECRET=$collab_secret/" \
  "$repo_root/compose/.env.example" > "$target"
chmod 600 "$target"
```

- [ ] **Step 4: Implement the canonical Compose wrapper plus stable start, stop, and smoke commands**

Create `ops/compose.sh`:

```sh
#!/bin/sh
set -eu

repo_root="$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)"
upstream="$repo_root/vendor/openproject-docker-compose/docker-compose.yml"
override="$repo_root/compose/docker-compose.override.yml"

exec docker compose \
  --env-file "$repo_root/compose/.env" \
  -f "$upstream" \
  -f "$override" \
  "$@"
```

Create `ops/up.sh`:

```sh
#!/bin/sh
set -eu
repo_root="$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)"
"$repo_root/ops/preflight.sh" --min-free-gib 30
docker context use colima-video-operations >/dev/null
"$repo_root/ops/compose.sh" up -d
```

Create `ops/down.sh`:

```sh
#!/bin/sh
set -eu
repo_root="$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)"
docker context use colima-video-operations >/dev/null
"$repo_root/ops/compose.sh" down
```

Create `ops/smoke.sh`:

```sh
#!/bin/sh
set -eu

health_url="http://localhost:8080/health_checks/default"
curl -fsS "$health_url" >/dev/null

listeners="$(lsof -nP -iTCP:8080 -sTCP:LISTEN)"
printf '%s\n' "$listeners" | grep '127.0.0.1:8080' >/dev/null
if printf '%s\n' "$listeners" | grep -E '(\*:8080|0\.0\.0\.0:8080)' >/dev/null; then
  printf '%s\n' 'OpenProject is exposed beyond loopback.' >&2
  exit 5
fi

printf '%s\n' 'OpenProject local smoke test passed.'
```

Run:

```bash
chmod +x ops/init-env.sh ops/compose.sh ops/up.sh ops/down.sh ops/smoke.sh
```

- [ ] **Step 5: Create the operator Makefile**

Create `Makefile`:

```makefile
.PHONY: preflight runtime init up down smoke test-ops

preflight:
	./ops/preflight.sh --min-free-gib 30

runtime:
	./ops/bootstrap-runtime.sh

init:
	./ops/init-env.sh

up:
	./ops/up.sh

down:
	./ops/down.sh

smoke:
	./ops/smoke.sh

test-ops:
	python3 -m unittest discover -s ops/tests -p 'test_*.py' -v
```

- [ ] **Step 6: Run the static tests**

Run:

```bash
make test-ops
```

Expected: all operations tests PASS.

- [ ] **Step 7: Create secrets and validate Compose before starting**

Run:

```bash
./ops/init-env.sh
./ops/compose.sh config --quiet
```

Expected: exit 0 and no Compose validation error.

- [ ] **Step 8: Start OpenProject and run the local smoke test**

Run:

```bash
make up
make smoke
```

Expected: `OpenProject local smoke test passed.` and the login page opens at `http://localhost:8080`.

- [ ] **Step 9: Verify pinned images and local listener**

Run:

```bash
./ops/compose.sh images
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

Expected: OpenProject images use `17.7.2-slim`, PostgreSQL uses `17`, and the only listener is `127.0.0.1:8080`.

- [ ] **Step 10: Commit**

```bash
git add compose/.env.example compose/docker-compose.override.yml ops/init-env.sh ops/compose.sh ops/up.sh ops/down.sh ops/smoke.sh Makefile ops/tests/test_local_compose.py
git commit -m "feat: run pinned OpenProject on localhost"
```

## Package Verification

Run from a clean checkout after initializing the submodule:

```bash
git submodule update --init --recursive
make test-ops
make preflight
./ops/compose.sh config --quiet
make up
make smoke
```

Expected:

- All Python tests pass.
- Compose configuration validates.
- OpenProject health check returns success.
- Port 8080 is loopback-only.
- `git status --short` is empty; `compose/.env` exists, has mode `0600`, and is ignored by Git.
