# Python Project Checklist (Short): pytest, Docker, GitHub Actions

Replace `YOUR-USERNAME` and `pytest-demo` with your own names.

```
pytest-demo/
├── .github/workflows/ci.yml
├── .dockerignore
├── .gitignore
├── Dockerfile            # runs the app
├── Dockerfile.test       # runs the tests
├── requirements.txt
├── example.py
├── main.py
├── test_example.py
├── run.sh
└── test.sh
```

> Your own structure may differ. These parts are necessary: both Dockerfiles (names must match what the scripts use), `requirements.txt` with `pytest`, test files named `test_*.py`, executable `run.sh`/`test.sh`, and `.github/workflows/ci.yml` at that exact path.

---

## 1. Prerequisites

- [ ] Docker Desktop is running: `docker version` shows a **Client** and a **Server**
- [ ] Git installed: `git --version`
- [ ] Windows: run `.sh` scripts in **Git Bash or WSL**

## 2. Create the Repo and Clone It

- [ ] GitHub → **+** → **New repository** → name `pytest-demo` → **Public** → check **Add a README** → **Create repository**
- [ ] `git clone https://github.com/YOUR-USERNAME/pytest-demo.git`
- [ ] `cd pytest-demo` then `code .`

## 3. Create the Files

- [ ] `mkdir -p .github/workflows`
- [ ] `touch .gitignore .dockerignore requirements.txt Dockerfile Dockerfile.test example.py main.py test_example.py run.sh test.sh`
- [ ] `.gitignore` and `.dockerignore` (same content in both, plus `.git` and `run.sh`/`test.sh` in the dockerignore):
```
__pycache__/
*.pyc
.pytest_cache/
venv/
.venv/
.env
.env.*
!.env.example
.DS_Store
```
Add these extra lines to `.dockerignore` only:
```
README.md
run.sh
test.sh
.git
.gitignore
```

## 4. Code and First Test

- [ ] `requirements.txt`:
```
pytest
```
- [ ] `example.py`:
```python
def add_two_numbers(num_one, num_two):
    return num_one + num_two
```
- [ ] `test_example.py`:
```python
from example import add_two_numbers

def test_add_two_numbers():
    assert add_two_numbers(2, 2) == 4
```
- [ ] `main.py`:
```python
from example import add_two_numbers

print(f"2 + 2 = {add_two_numbers(2, 2)}")
```

## 5. Dockerfiles

- [ ] `Dockerfile` (runs the app):
```
FROM python:3.11-slim

WORKDIR /app

COPY . .

CMD ["python3", "main.py"]
```
- [ ] `Dockerfile.test` (runs the tests):
```
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "-m", "pytest"]
```

## 6. Scripts

- [ ] `test.sh`:
```bash
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")"

echo "Building test image..."
docker build -t pytest-env -f Dockerfile.test .

echo "Running tests..."
docker run --rm pytest-env

echo "Done! Tests passed."
```
- [ ] `run.sh`:
```bash
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")"

echo "Building app image..."
docker build -t pytest-app .

echo "Running app..."
docker run --rm --init pytest-app
```
- [ ] `chmod +x run.sh test.sh`
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` shows `1 passed`, and `echo $?` prints **0**

> Images are snapshots, so edit code, then rebuild. The scripts rebuild every time.

## 7. Push `main`, Create `dev`

- [ ] `git add .`
- [ ] `git commit -m "chore: project scaffold"`
- [ ] `git push origin main`
- [ ] `git checkout -b dev`
- [ ] `git push -u origin dev`

## 8. Add GitHub Actions

- [ ] `git checkout -b feature/ci`
- [ ] Create `.github/workflows/ci.yml` (spaces only, never tabs):
```yaml
name: Python CI Pipeline

on:
  push:
    branches:
      - main
      - dev
  pull_request:
    branches:
      - main
      - dev
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Run pytest in Docker
        run: bash test.sh
```
This runs `test.sh` on a fresh Ubuntu machine on every push to `main`/`dev` and every PR into them. Checkout must be the first step. Check the action's Releases page for the newest `checkout` version.
- [ ] `git add .`
- [ ] `git commit -m "ci: run pytest in Docker"`
- [ ] `git push -u origin feature/ci`
- [ ] On GitHub, open a PR **into `dev`**, wait for the green **test** check ✓, merge, then:
  - `git checkout dev`
  - `git pull origin dev`
  - `git branch -d feature/ci`

## 9. Prove It Goes Red, Then Green

- [ ] `git checkout -b try-red`, change `== 4` to `== 5` in `test_example.py`
- [ ] `./test.sh` fails (exit code **1**)
- [ ] Commit, push, open a PR into `dev`. The **test** check shows a red ✗
- [ ] Change it back, push. The check turns green ✓. Then close the PR and delete the branch.

## 10. TDD: Red, Green, Refactor

Workflow for every feature: branch off `dev`, work, push, PR into `dev`, merge. Release by merging `dev` into `main`.

- [ ] `git checkout dev && git pull origin dev`
- [ ] `git checkout -b feature/factorial`
- [ ] **RED:** create `test_factorial.py`:
```python
from factorial import factorial

def test_factorial_of_zero():
    assert factorial(0) == 1

def test_factorial_of_five():
    assert factorial(5) == 120
```
- [ ] `./test.sh` fails with an **ImportError** (exit code 2) 🔴
- [ ] **GREEN:** create `factorial.py`:
```python
def factorial(num):
    product = 1
    for i in range(num, 0, -1):
        product = product * i
    return product
```
- [ ] `./test.sh` shows `3 passed` 🟢
- [ ] **REFACTOR:** simplify (e.g. `math.prod(range(1, num + 1))`), then run `./test.sh` again
- [ ] `git add .`
- [ ] `git commit -m "feat: factorial kata"`
- [ ] `git push -u origin feature/factorial`
- [ ] PR into `dev`, wait for green, merge. Then PR `dev` into `main` to release.

---

## Troubleshooting

| You see | Fix |
|---|---|
| `Cannot connect to the Docker daemon` | Open Docker Desktop and wait |
| `Permission denied` on a script | `chmod +x run.sh test.sh` |
| `failed to read dockerfile` | Run from the repo root; check for `Dockerfile.txt` |
| `no tests ran` (exit 5) | Name files `test_*.py` and functions `test_*` |
| No Actions run after pushing | Workflow must be at `.github/workflows/*.yml`; the branch must be listed under `on:` |
| `Invalid workflow file ... yaml syntax` | Use spaces, not tabs |
| Dockerfile not found in CI | `actions/checkout` must be the first step |
