# Python Project Checklist: pytest, Docker, GitHub Actions

Goal: a GitHub repo with a Python project tested and run inside Docker, driven by `run.sh` and `test.sh`, and verified automatically by GitHub Actions on every push and pull request.

Replace `YOUR-USERNAME` and `pytest-demo` (the repo name) wherever you see them.

## Final Folder Structure

```
pytest-demo/
├── .github/
│   └── workflows/
│       └── ci.yml
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

---

## 0. Before You Start

- [ ] Docker Desktop is open and running (Mac/Windows). Check it:
  - `docker version` (you must see both a **Client** and a **Server** section)
  - `docker run --rm hello-world`
- [ ] Git is installed: `git --version`
- [ ] Windows users: run `.sh` scripts from **Git Bash or WSL**, not PowerShell or Command Prompt
- [ ] Pre-pull the base image: `docker pull python:3.11-slim`
- [ ] You do **not** need Python installed locally. Docker brings it.

---

## 1. Create the Remote Repository on GitHub

- [ ] Go to [GitHub](https://github.com), click **"+"** (top right) → **New repository**
- [ ] Name it (e.g., `pytest-demo`)
- [ ] Choose **Public** (recommended: branch protection is free on public repos) or **Private**
- [ ] Check **Add a README file**. Leave **.gitignore** and **license** unset (we write our own)
- [ ] Click **Create repository**
- [ ] Copy the clone URL from the green **Code** button

---

## 2. Clone It Down to Local

- [ ] `git clone <GITHUB_REPO_URL>`
- [ ] `cd pytest-demo`
- [ ] `code .` (no `code` command? VS Code → **File → Open Folder**)
- [ ] Confirm you're on `main` with the remote set:
  - `git branch`
  - `git remote -v`

> **Starting from a local folder instead?** `git init -b main`, then `git remote add origin <URL>`. If you get `remote origin already exists`, use `git remote set-url origin <URL>`.

---

## 3. Create the Baseline Files

- [ ] From the repo root:
  - `mkdir -p .github/workflows`
  - `touch .gitignore .dockerignore requirements.txt Dockerfile Dockerfile.test example.py main.py test_example.py run.sh test.sh`
- [ ] Check with `ls -a`. Dockerfiles must be named exactly `Dockerfile` (capital D, no `.txt`).

> **Your own file structure will probably be different.** File names like `example.py` and `main.py`, and whether you keep code in the root or in folders like `src/` and `tests/`, are up to you. If you rename or move things, update every place that points to them (the `Dockerfile` `CMD`, the imports in your tests, and the scripts).
>
> **These parts are necessary no matter what:**
> - A `Dockerfile` and a `Dockerfile.test` (or whatever names you choose, as long as `run.sh` and `test.sh` use the same names with `-f`)
> - `requirements.txt` listing `pytest`, which the test image installs
> - Test files named `test_*.py`, with test functions named `test_*`. Otherwise pytest finds nothing (exit code 5).
> - `run.sh` and `test.sh` at the repo root, made executable with `chmod +x`
> - `.github/workflows/ci.yml` at exactly that path (a `.yml` file inside `.github/workflows/`)
> - A `.gitignore` and `.dockerignore` so junk stays out of Git and the image

---

## 4. Configure `.gitignore`

- [ ] Keep environment artifacts out of Git:
```
# Python / Pytest
__pycache__/
*.pyc
.pytest_cache/
venv/
.venv/

# Docker / environment
.env
.env.*
!.env.example

# OS
.DS_Store
```

## 5. Configure `.dockerignore`

- [ ] Keep local docs, scripts, and Git data out of the image:
```
# Documentation & local scripts
README.md
run.sh
test.sh

# Git artifacts
.git
.gitignore

# Python artifacts
__pycache__/
.pytest_cache/
venv/
.venv/
```

---

## 6. Write the Code and the First Test

A tiny starter proves the whole pipeline works before you do real TDD.

- [ ] `requirements.txt`:
```
pytest
```
- [ ] `example.py`:
```python
def add_two_numbers(num_one, num_two):
    return num_one + num_two
```
- [ ] `test_example.py` (test files and functions **must** start with `test_`):
```python
from example import add_two_numbers

def test_add_two_numbers():
    assert add_two_numbers(2, 2) == 4
```
- [ ] `main.py` (what the "run" image executes):
```python
from example import add_two_numbers

print(f"2 + 2 = {add_two_numbers(2, 2)}")
```

---

## 7. Dockerfiles

### 7a. Run image
- [ ] `Dockerfile`:
```
FROM python:3.11-slim

WORKDIR /app

COPY . .

CMD ["python3", "main.py"]
```

### 7b. Test image
- [ ] `Dockerfile.test`:
```
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "-m", "pytest"]
```
- `RUN` happens at **build** time (installs pytest). `CMD` happens at **run** time (runs the suite).
- Copying `requirements.txt` first lets Docker reuse the cached install layer when only your code changes.
- Pin the version (`python:3.11-slim`), never plain `python` (that means `latest`).

---

## 8. Shell Scripts

### 8a. `test.sh`
- [ ] `test.sh`:
```bash
#!/usr/bin/env bash

# Exit immediately if any command fails (so CI sees a failure)
set -e

cd "$(dirname "$0")"

IMAGE_NAME="pytest-env"

echo "----------------------------------------"
echo "Building Docker image: $IMAGE_NAME..."
echo "----------------------------------------"
docker build -t "$IMAGE_NAME" -f Dockerfile.test .

echo "----------------------------------------"
echo "Running containerized tests..."
echo "----------------------------------------"
docker run --rm "$IMAGE_NAME"

echo "----------------------------------------"
echo "Done! Tests passed."
echo "----------------------------------------"
```

### 8b. `run.sh`
- [ ] `run.sh`:
```bash
#!/usr/bin/env bash

# Exit immediately if any command fails
set -e

cd "$(dirname "$0")"

IMAGE_NAME="pytest-app"

echo "Building Docker image: $IMAGE_NAME..."
docker build -t "$IMAGE_NAME" .

echo "Running the app..."
docker run --rm --init "$IMAGE_NAME"

echo "Done!"
```

### 8c. Make them executable and try them
- [ ] `chmod +x run.sh test.sh` (without this you get `Permission denied`)
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` shows `1 passed`
- [ ] `echo $?` prints **0**
- [ ] `docker ps -a` shows no leftover containers (`--rm` cleans up)

**Flags:** `-t` names the image, `-f` picks the Dockerfile, the final `.` is the build context (this folder), `--rm` deletes the container on exit, `--init` lets Ctrl+C stop a process in a container.

> Images are snapshots. **Edit code, then rebuild.** The scripts rebuild every time.

---

## 9. Commit and Push `main`

- [ ] `git status`
- [ ] `git add .`
- [ ] `git commit -m "chore: project scaffold with Docker and pytest"`
- [ ] `git push origin main`

---

## 10. Branching Workflow

- [ ] Create and publish a long-lived integration branch:
  - `git checkout -b dev`
  - `git push -u origin dev`
- [ ] For each new piece of work, branch off an up-to-date `dev`:
  - `git checkout dev`
  - `git pull origin dev`
  - `git checkout -b feature/<name>`
- [ ] Work, run `./test.sh`, commit, and push:
  - `git add .`
  - `git commit -m "feat: <what you did>"`
  - `git push -u origin feature/<name>`
- [ ] On GitHub, open a **Pull Request** from `feature/<name>` **into `dev`**
- [ ] Wait for the CI check to go green (after step 11), then **Merge pull request**
- [ ] Clean up locally:
  - `git checkout dev`
  - `git pull origin dev`
  - `git branch -d feature/<name>`
- [ ] When `dev` is ready for release, open a PR from `dev` **into `main`** and merge

---

## 11. Add GitHub Actions (CI)

Do this on its own feature branch so your first PR runs the pipeline.

- [ ] `git checkout dev && git pull origin dev`
- [ ] `git checkout -b feature/ci`
- [ ] Create `.github/workflows/ci.yml` (`.github` with a dot, `workflows` with an **s**, ends in `.yml`; **spaces only, never tabs**):
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
- [ ] `git add .`
- [ ] `git commit -m "ci: run pytest in Docker on push and PR"`
- [ ] `git push -u origin feature/ci`
- [ ] On GitHub, open a PR **into `dev`**. At the bottom you'll see a **test** check. Wait for green ✓.
- [ ] Merge, delete the branch, pull `dev` locally

| Line | Means |
|---|---|
| `on: push: branches` | Run when someone pushes to `main` or `dev` |
| `on: pull_request: branches` | Run on PRs **targeting** `main` or `dev` |
| `workflow_dispatch:` | Adds a **Run workflow** button in the Actions tab |
| `jobs: test:` | One job. Its id is the check name on PRs |
| `runs-on: ubuntu-latest` | A fresh Ubuntu machine each run (Docker pre-installed) |
| `actions/checkout` | Copies your repo onto the runner. **Without it nothing else works.** |
| `run: bash test.sh` | The exact script you run locally |

Notes:
- `@v7` matches your Code Lab 2 (the older curriculum shows `@v4`). I haven't verified that version, so check the action's **Releases** page. A yellow "deprecated" warning means an old version, not a failed run.
- Branches not listed under `on:` never trigger it. A push to `feature/ci` doesn't run, but the PR into `dev` does.
- **Alternative without Docker:** replace the run step with `actions/setup-python` (python-version `3.11`), then `pip install -r requirements.txt`, then `python -m pytest`.

---

## 12. Verify the Pipeline (Red, Then Green)

- [ ] On a throwaway branch (`git checkout -b try-red`), change `== 4` to `== 5` in `test_example.py`
- [ ] `./test.sh` should fail with an `AssertionError` and exit code **1**
- [ ] Commit, push, open a PR into `dev`. The **test** check should show a red ✗
- [ ] Change it back, commit, push. The check turns green ✓
- [ ] Close or merge the PR, delete the branch
- [ ] **Actions** → **Python CI Pipeline** → **Run workflow** to start a run by hand

**Optional: block merges when CI is red.**
- [ ] **Settings** → **Branches** → **Add classic branch protection rule**
- [ ] Branch name pattern: `main` (repeat for `dev` if you like)
- [ ] Tick **Require status checks to pass before merging** and add **test** (it only appears after running once)
- [ ] Tick **Do not allow bypassing the above settings**
- [ ] **Create**. Free on public repos; private repos need a paid plan.

---

## 13. Red → Green → Refactor (TDD)

Write the test first and watch it fail, write just enough code to pass, then clean up.

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
- [ ] `./test.sh`. You should see an **ImportError** (no `factorial` yet) and exit code **2**. 🔴
- [ ] **GREEN:** create `factorial.py`:
```python
def factorial(num):
    product = 1
    for i in range(num, 0, -1):
        product = product * i
    return product
```
- [ ] `./test.sh`. You should see `3 passed` (the example plus these two), exit code **0**. 🟢
- [ ] **REFACTOR:** simplify while staying green, e.g. `return math.prod(range(1, num + 1))` (add `import math`). Run `./test.sh` again.
- [ ] `git add .` → `git commit -m "feat: factorial kata"` → `git push -u origin feature/factorial`
- [ ] PR **into `dev`**, confirm **test** is green, merge, delete the branch
- [ ] Release: PR from `dev` **into `main`**, confirm green, merge

**Exit codes:** **0** all passed · **1** a test failed · **2** pytest couldn't collect (e.g. ImportError) · **5** no tests found. CI only looks at this number.

**pytest tools:** `pytest.raises(ZeroDivisionError)` for expected errors, `monkeypatch` to fake `input()`, `capsys` to capture `print()`, `math.isclose(a, b)` for floats.

---

## 14. Cleanup

- [ ] See leftovers: `docker ps -a`, `docker images`
- [ ] Remove a stopped container: `docker rm <name>` (running: `docker rm -f <name>`)
- [ ] Remove an image: `docker rmi <image>` (containers first, then images)
- [ ] Remove untagged images: `docker image prune`

---

## Troubleshooting

| You see | It means | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop isn't running | Open it and wait |
| `./test.sh: Permission denied` | Script isn't executable | `chmod +x run.sh test.sh` |
| `failed to read dockerfile` | Wrong folder, or file named `Dockerfile.txt` | Run from the repo root; check `ls` |
| `'docker buildx build' requires 1 argument` | Missing the `.` at the end | Add the build context `.` |
| Old output after editing code | Image has the old copy | Rebuild (scripts do this) |
| `no tests ran` (exit 5) | pytest can't find tests | Files `test_*.py`, functions `def test_...()` |
| `ModuleNotFoundError: No module named 'example'` | Running from the wrong folder | Run via `./test.sh` from the repo root |
| `ImportError: cannot import name ...` | Function doesn't exist yet | In TDD this is your first red |
| `assert 0.30000000000000004 == 0.3` | Float rounding | `math.isclose(a, b)` |
| No Actions run after pushing | Wrong path, or branch not listed under `on:` | Check `.github/workflows/*.yml` and branches |
| `Invalid workflow file ... yaml syntax` | Tab or bad indentation | Spaces only |
| `failed to read dockerfile` in CI | No checkout step | `actions/checkout@v7` must be first |
| No **Run workflow** button | `workflow_dispatch:` missing or not on `main` yet | Add it, merge to `main` |
| `src refspec main does not match any` | Nothing committed yet | `git add .` then `git commit` |
| Actions tab missing | Actions disabled | **Settings → Actions → General → Allow all actions** |

---

## Definition of Done

- [ ] Repo exists on GitHub and is cloned locally
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` runs pytest in Docker and exits **0**
- [ ] You saw a test **fail first** (red) before it passed (green)
- [ ] `ci.yml` is on `dev` and `main`
- [ ] A PR into `dev` shows a green **test** check
- [ ] You've seen one red ✗ run in Actions, then a green ✓
- [ ] Work flows `feature/*` → `dev` → `main`

---

## Source Material

Fundamentals Week: Docker Lectures 1-2, JS & Python Lectures 1-2, TDD Lecture 1 (pytest), GitHub Actions Lectures 1-2.
