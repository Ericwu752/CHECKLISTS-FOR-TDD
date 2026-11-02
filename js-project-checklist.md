# JavaScript Project Checklist: Jest, Docker, GitHub Actions

Goal: a GitHub repo with a JavaScript project tested and run inside Docker, driven by `run.sh` and `test.sh`, and verified automatically by GitHub Actions on every push and pull request.

Replace `YOUR-USERNAME` and `jest-demo` (the repo name) wherever you see them.

## Final Folder Structure

```
jest-demo/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .dockerignore
├── .gitignore
├── Dockerfile            # runs the app
├── Dockerfile.test       # runs the tests
├── package.json
├── package-lock.json
├── example.js
├── index.js
├── example.test.js
├── run.sh
└── test.sh
```

---

## 0. Before You Start

- [ ] Docker Desktop is open and running (Mac/Windows). Check it:
  - `docker version` (you must see both a **Client** and a **Server** section)
  - `docker run --rm hello-world`
- [ ] Git is installed: `git --version`
- [ ] Node 18+ is installed for the one-time `npm install` in step 6: `node --version`. (No Node? There's a Docker-only alternative in step 6.)
- [ ] Windows users: run `.sh` scripts from **Git Bash or WSL**, not PowerShell or Command Prompt
- [ ] Pre-pull the base image: `docker pull node:20-alpine`

---

## 1. Create the Remote Repository on GitHub

- [ ] Go to [GitHub](https://github.com), click **"+"** (top right) → **New repository**
- [ ] Name it (e.g., `jest-demo`)
- [ ] Choose **Public** (recommended: branch protection is free on public repos) or **Private**
- [ ] Check **Add a README file**. Leave **.gitignore** and **license** unset (we write our own)
- [ ] Click **Create repository**
- [ ] Copy the clone URL from the green **Code** button

---

## 2. Clone It Down to Local

- [ ] `git clone <GITHUB_REPO_URL>`
- [ ] `cd jest-demo`
- [ ] `code .` (no `code` command? VS Code → **File → Open Folder**)
- [ ] Confirm you're on `main` with the remote set:
  - `git branch`
  - `git remote -v`

> **Starting from a local folder instead?** `git init -b main`, then `git remote add origin <URL>`. If you get `remote origin already exists`, use `git remote set-url origin <URL>`.

---

## 3. Create the Baseline Files

- [ ] From the repo root:
  - `mkdir -p .github/workflows`
  - `touch .gitignore .dockerignore Dockerfile Dockerfile.test example.js index.js example.test.js run.sh test.sh`
- [ ] Check with `ls -a`. Dockerfiles must be named exactly `Dockerfile` (capital D, no `.txt`).

> **Your own file structure will probably be different.** File names like `example.js` and `index.js`, and whether you keep code in the root or in folders like `src/` and `tests/`, are up to you. If you rename or move things, update every place that points to them (the `Dockerfile` `CMD`, the `require` paths in your tests, the `main` field in `package.json`, and the scripts).
>
> **These parts are necessary no matter what:**
> - A `Dockerfile` and a `Dockerfile.test` (or whatever names you choose, as long as `run.sh` and `test.sh` use the same names with `-f`)
> - `package.json` with `"test": "jest"` and Jest in `devDependencies`, plus a committed `package-lock.json` (the test image runs `npm ci`)
> - Test files named `*.test.js` or `*.spec.js`. Otherwise Jest finds nothing.
> - `module.exports` in code files and `require` in tests, so tests can load your code
> - `run.sh` and `test.sh` at the repo root, made executable with `chmod +x`
> - `.github/workflows/ci.yml` at exactly that path (a `.yml` file inside `.github/workflows/`)
> - A `.gitignore` and `.dockerignore` (with `node_modules` in both) so junk stays out of Git and the image

---

## 4. Configure `.gitignore`

- [ ] Keep environment artifacts out of Git:
```
# Node / JavaScript
node_modules/
dist/
dist-ssr/
build/
coverage/
*.tsbuildinfo
.vite/

# Docker / environment
.env
.env.*
!.env.example

# OS
.DS_Store
```

> Do **not** ignore `package-lock.json`. The test image installs from it.

## 5. Configure `.dockerignore`

- [ ] So your laptop's `node_modules` never overwrites the container's:
```
# Dependencies & build output
node_modules
coverage

# Documentation & local scripts
README.md
run.sh
test.sh

# Git artifacts
.git
.gitignore
```

---

## 6. Set Up `package.json` and Jest

- [ ] `npm init -y`
- [ ] In `package.json`, change the test script to:
```json
"scripts": {
  "test": "jest"
}
```
- [ ] Install Jest (this also creates `package-lock.json`):
  - `npm install --save-dev jest`
- [ ] Confirm `package.json` lists `jest` under `devDependencies`

> **No Node installed?** Use Docker (from the repo root). On Linux the files may end up owned by root.
> `docker run --rm -v "$PWD":/app -w /app node:20-alpine sh -c "npm init -y && npm install --save-dev jest"`
> (Git Bash: add `MSYS_NO_PATHCONV=1` before `docker`. PowerShell: use `${PWD}`.) You still edit the `"test"` script by hand.

---

## 7. Write the Code and the First Test

A tiny starter proves the whole pipeline works before you do real TDD.

- [ ] `example.js` (it needs `module.exports`, or tests can't `require` it):
```javascript
function addTwoNumbers(numOne, numTwo) {
  return numOne + numTwo;
}

module.exports = addTwoNumbers;
```
- [ ] `example.test.js` (Jest finds files ending in `.test.js` or `.spec.js`):
```javascript
const addTwoNumbers = require("./example.js");

test("adds 2 + 2 = 4", () => {
  expect(addTwoNumbers(2, 2)).toBe(4);
});
```
- [ ] `index.js` (what the "run" image executes):
```javascript
const addTwoNumbers = require("./example.js");

console.log(`2 + 2 = ${addTwoNumbers(2, 2)}`);
```
- [ ] Sanity check locally: `node index.js` prints `2 + 2 = 4`, and `npm test` shows `1 passed`

---

## 8. Dockerfiles

### 8a. Run image
- [ ] `Dockerfile`:
```
FROM node:20-alpine

WORKDIR /app

COPY . .

CMD ["node", "index.js"]
```

### 8b. Test image
- [ ] `Dockerfile.test`:
```
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

CMD ["npm", "test"]
```
- `RUN` happens at **build** time (installs Jest). `CMD` happens at **run** time (runs the suite).
- Copying `package*.json` first lets Docker reuse the cached install layer when only your code changes.
- `npm ci` installs exactly what `package-lock.json` says. The class labs use `npm install` instead, and either works.
- The labs use `node:18-alpine`. Node 20 just avoids Jest's `EBADENGINE` warnings. Always pin the version, never plain `node`.

---

## 9. Shell Scripts

### 9a. `test.sh`
- [ ] `test.sh`:
```bash
#!/usr/bin/env bash

# Exit immediately if any command fails (so CI sees a failure)
set -e

cd "$(dirname "$0")"

IMAGE_NAME="jest-env"

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

### 9b. `run.sh`
- [ ] `run.sh`:
```bash
#!/usr/bin/env bash

# Exit immediately if any command fails
set -e

cd "$(dirname "$0")"

IMAGE_NAME="jest-app"

echo "Building Docker image: $IMAGE_NAME..."
docker build -t "$IMAGE_NAME" .

echo "Running the app..."
docker run --rm --init "$IMAGE_NAME"

echo "Done!"
```

### 9c. Make them executable and try them
- [ ] `chmod +x run.sh test.sh` (without this you get `Permission denied`)
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` shows `Tests: 1 passed, 1 total`
- [ ] `echo $?` prints **0**
- [ ] `docker ps -a` shows no leftover containers (`--rm` cleans up)

**Flags:** `-t` names the image, `-f` picks the Dockerfile, the final `.` is the build context (this folder), `--rm` deletes the container on exit, `--init` lets Ctrl+C stop Node inside a container.

> Images are snapshots. **Edit code, then rebuild.** The scripts rebuild every time.

---

## 10. Commit and Push `main`

- [ ] `git status` (check that `node_modules/` is **not** listed, and `package-lock.json` **is**)
- [ ] `git add .`
- [ ] `git commit -m "chore: project scaffold with Docker and Jest"`
- [ ] `git push origin main`

---

## 11. Branching Workflow

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
- [ ] Wait for the CI check to go green (after step 12), then **Merge pull request**
- [ ] Clean up locally:
  - `git checkout dev`
  - `git pull origin dev`
  - `git branch -d feature/<name>`
- [ ] When `dev` is ready for release, open a PR from `dev` **into `main`** and merge

---

## 12. Add GitHub Actions (CI)

Do this on its own feature branch so your first PR runs the pipeline.

- [ ] `git checkout dev && git pull origin dev`
- [ ] `git checkout -b feature/ci`
- [ ] Create `.github/workflows/ci.yml` (`.github` with a dot, `workflows` with an **s**, ends in `.yml`; **spaces only, never tabs**):
```yaml
name: JS CI Pipeline

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

      - name: Run Jest in Docker
        run: bash test.sh
```
- [ ] `git add .`
- [ ] `git commit -m "ci: run Jest in Docker on push and PR"`
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
- **Alternative without Docker** (from Code Lab 2): replace the run step with `actions/setup-node@v7` (`node-version: 20`), then `npm ci`, then `npm test`.

---

## 13. Verify the Pipeline (Red, Then Green)

- [ ] On a throwaway branch (`git checkout -b try-red`), change `toBe(4)` to `toBe(5)` in `example.test.js`
- [ ] `./test.sh` should fail with `Expected: 5 / Received: 4` and exit code **1**
- [ ] Commit, push, open a PR into `dev`. The **test** check should show a red ✗ (`Process completed with exit code 1`)
- [ ] Change it back, commit, push. The check turns green ✓
- [ ] Close or merge the PR, delete the branch
- [ ] **Actions** → **JS CI Pipeline** → **Run workflow** to start a run by hand

**Optional: block merges when CI is red.**
- [ ] **Settings** → **Branches** → **Add classic branch protection rule**
- [ ] Branch name pattern: `main` (repeat for `dev` if you like)
- [ ] Tick **Require status checks to pass before merging** and add **test** (it only appears after running once)
- [ ] Tick **Do not allow bypassing the above settings**
- [ ] **Create**. Free on public repos; private repos need a paid plan.

---

## 14. Red → Green → Refactor (TDD)

Write the test first and watch it fail, write just enough code to pass, then clean up.

- [ ] `git checkout dev && git pull origin dev`
- [ ] `git checkout -b feature/factorial`
- [ ] **RED:** create `factorial.test.js`:
```javascript
const factorial = require("./factorial.js");

describe("factorial", () => {
  test("factorial(0) = 1", () => {
    expect(factorial(0)).toBe(1);
  });

  test("factorial(5) = 120", () => {
    expect(factorial(5)).toBe(120);
  });
});
```
- [ ] `./test.sh`. You should see `Cannot find module './factorial.js'` and exit code **1**. 🔴
- [ ] **GREEN:** create `factorial.js`:
```javascript
function factorial(num) {
  let product = 1;

  for (let i = num; i > 0; i--) {
    product = product * i;
  }

  return product;
}

module.exports = factorial;
```
- [ ] `./test.sh`. You should see `Tests: 3 passed` (the example plus these two), exit code **0**. 🟢
- [ ] **REFACTOR:** simplify while staying green (e.g., recursion or `reduce`). Run `./test.sh` again.
- [ ] `git add .` → `git commit -m "feat: factorial kata"` → `git push -u origin feature/factorial`
- [ ] PR **into `dev`**, confirm **test** is green, merge, delete the branch
- [ ] Release: PR from `dev` **into `main`**, confirm green, merge

**Jest tools:** `describe` groups tests, `xtest` / `xdescribe` skip them, `toEqual` for arrays/objects (`toBe` checks they're the *same* object). Exit code **0** means pass, **1** means fail, and CI only looks at this number.

---

## 15. Optional: Fast Local Loop (Bind Mount)

`COPY` freezes code at build time. A bind mount gives the container a live view of your folder, so no rebuild.

- [ ] `docker run --rm -v "$PWD":/app -v /app/node_modules jest-env`
- [ ] The second `-v` keeps the container's own `node_modules` instead of hiding it behind your folder
- [ ] Use `COPY` for shipping and CI, a bind mount for daily development
- [ ] Git Bash: put `MSYS_NO_PATHCONV=1` before `docker`. PowerShell: `-v "${PWD}:/app"`

## 16. Cleanup

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
| `sh: jest: not found` | Jest isn't in `package.json` | `npm install --save-dev jest`, commit `package.json` **and** `package-lock.json` |
| `npm ci` fails: no `package-lock.json` | Lock file missing or gitignored | Run `npm install`, commit the lock file |
| `Error: no test specified` | `package.json` still has the default test script | `"test": "jest"` |
| `No tests found, exiting with code 1` | Jest needs `*.test.js` or `*.spec.js` | Rename the file |
| `ReferenceError: ... is not defined` | Missing `require` | `const fn = require("./file.js");` |
| `TypeError: ... is not a function` | Missing `module.exports` | Add `module.exports = fn;` at the bottom |
| `Received: serializes to the same string` | `toBe` on arrays/objects | Use `toEqual` |
| No Actions run after pushing | Wrong path, or branch not listed under `on:` | Check `.github/workflows/*.yml` and branches |
| `Invalid workflow file ... yaml syntax` | Tab or bad indentation | Spaces only |
| `npm error ENOENT ... package.json` in CI | No checkout step | `actions/checkout@v7` must be first |
| No **Run workflow** button | `workflow_dispatch:` missing or not on `main` yet | Add it, merge to `main` |
| `src refspec main does not match any` | Nothing committed yet | `git add .` then `git commit` |
| Actions tab missing | Actions disabled | **Settings → Actions → General → Allow all actions** |

---

## Definition of Done

- [ ] Repo exists on GitHub and is cloned locally
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` runs Jest in Docker and exits **0**
- [ ] You saw a test **fail first** (red) before it passed (green)
- [ ] `package-lock.json` is committed
- [ ] `ci.yml` is on `dev` and `main`
- [ ] A PR into `dev` shows a green **test** check
- [ ] You've seen one red ✗ run in Actions, then a green ✓
- [ ] Work flows `feature/*` → `dev` → `main`

---

## Source Material

Fundamentals Week: Docker Lectures 1-2, JS & Python Lectures 1-2, TDD Lecture 2 (Jest), GitHub Actions Lectures 1-2.
