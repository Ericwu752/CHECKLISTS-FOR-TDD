# JavaScript Project Checklist (Short): Jest, Docker, GitHub Actions

Replace `YOUR-USERNAME` and `jest-demo` with your own names.

```
jest-demo/
├── .github/workflows/ci.yml
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

> Your own structure may differ. These parts are necessary: both Dockerfiles (names must match what the scripts use), `package.json` with `"test": "jest"` and Jest as a dev dependency, a committed `package-lock.json`, test files named `*.test.js`, `module.exports`/`require` so tests can load code, executable `run.sh`/`test.sh`, and `.github/workflows/ci.yml` at that exact path.

---

## 1. Prerequisites

- [ ] Docker Desktop is running: `docker version` shows a **Client** and a **Server**
- [ ] Git installed: `git --version`
- [ ] Node 18+ for the one-time `npm install`: `node --version`
- [ ] Windows: run `.sh` scripts in **Git Bash or WSL**

## 2. Create the Repo and Clone It

- [ ] GitHub → **+** → **New repository** → name `jest-demo` → **Public** → check **Add a README** → **Create repository**
- [ ] `git clone https://github.com/YOUR-USERNAME/jest-demo.git`
- [ ] `cd jest-demo` then `code .`

## 3. Create the Files

- [ ] `mkdir -p .github/workflows`
- [ ] `touch .gitignore .dockerignore Dockerfile Dockerfile.test example.js index.js example.test.js run.sh test.sh`
- [ ] `.gitignore`:
```
node_modules/
dist/
build/
coverage/
.env
.env.*
!.env.example
.DS_Store
```
Do **not** ignore `package-lock.json`.
- [ ] `.dockerignore` (keeps your laptop's `node_modules` out of the image):
```
node_modules
coverage
README.md
run.sh
test.sh
.git
.gitignore
```

## 4. Set Up Jest

- [ ] `npm init -y`
- [ ] In `package.json`, set the test script:
```json
"scripts": {
  "test": "jest"
}
```
- [ ] `npm install --save-dev jest` (creates `package-lock.json`)

## 5. Code and First Test

- [ ] `example.js`:
```javascript
function addTwoNumbers(numOne, numTwo) {
  return numOne + numTwo;
}

module.exports = addTwoNumbers;
```
- [ ] `example.test.js`:
```javascript
const addTwoNumbers = require("./example.js");

test("adds 2 + 2 = 4", () => {
  expect(addTwoNumbers(2, 2)).toBe(4);
});
```
- [ ] `index.js`:
```javascript
const addTwoNumbers = require("./example.js");

console.log(`2 + 2 = ${addTwoNumbers(2, 2)}`);
```
- [ ] Sanity check: `node index.js` prints `2 + 2 = 4`, and `npm test` shows `1 passed`

## 6. Dockerfiles

- [ ] `Dockerfile` (runs the app):
```
FROM node:20-alpine

WORKDIR /app

COPY . .

CMD ["node", "index.js"]
```
- [ ] `Dockerfile.test` (runs the tests):
```
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

CMD ["npm", "test"]
```

## 7. Scripts

- [ ] `test.sh`:
```bash
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")"

echo "Building test image..."
docker build -t jest-env -f Dockerfile.test .

echo "Running tests..."
docker run --rm jest-env

echo "Done! Tests passed."
```
- [ ] `run.sh`:
```bash
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")"

echo "Building app image..."
docker build -t jest-app .

echo "Running app..."
docker run --rm --init jest-app
```
- [ ] `chmod +x run.sh test.sh`
- [ ] `./run.sh` prints `2 + 2 = 4`
- [ ] `./test.sh` shows `Tests: 1 passed`, and `echo $?` prints **0**

> Images are snapshots, so edit code, then rebuild. The scripts rebuild every time.

## 8. Push `main`, Create `dev`

- [ ] `git status` (`node_modules/` should **not** appear; `package-lock.json` should)
- [ ] `git add .`
- [ ] `git commit -m "chore: project scaffold"`
- [ ] `git push origin main`
- [ ] `git checkout -b dev`
- [ ] `git push -u origin dev`

## 9. Add GitHub Actions

- [ ] `git checkout -b feature/ci`
- [ ] Create `.github/workflows/ci.yml` (spaces only, never tabs):
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
This runs `test.sh` on a fresh Ubuntu machine on every push to `main`/`dev` and every PR into them. Checkout must be the first step. Check the action's Releases page for the newest `checkout` version.
- [ ] `git add .`
- [ ] `git commit -m "ci: run Jest in Docker"`
- [ ] `git push -u origin feature/ci`
- [ ] On GitHub, open a PR **into `dev`**, wait for the green **test** check ✓, merge, then:
  - `git checkout dev`
  - `git pull origin dev`
  - `git branch -d feature/ci`

## 10. Prove It Goes Red, Then Green

- [ ] `git checkout -b try-red`, change `toBe(4)` to `toBe(5)` in `example.test.js`
- [ ] `./test.sh` fails with `Expected: 5 / Received: 4` (exit code **1**)
- [ ] Commit, push, open a PR into `dev`. The **test** check shows a red ✗
- [ ] Change it back, push. The check turns green ✓. Then close the PR and delete the branch.

## 11. TDD: Red, Green, Refactor

Workflow for every feature: branch off `dev`, work, push, PR into `dev`, merge. Release by merging `dev` into `main`.

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
- [ ] `./test.sh` fails with `Cannot find module './factorial.js'` 🔴
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
- [ ] `./test.sh` shows `Tests: 3 passed` 🟢
- [ ] **REFACTOR:** simplify (e.g. recursion or `reduce`), then run `./test.sh` again
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
| `sh: jest: not found` / `npm ci` fails | `npm install --save-dev jest`; commit `package.json` **and** `package-lock.json` |
| `No tests found` | Name files `*.test.js` or `*.spec.js` |
| `... is not a function` / `not defined` | Add `module.exports` in the code file and `require` in the test |
| No Actions run after pushing | Workflow must be at `.github/workflows/*.yml`; the branch must be listed under `on:` |
| `Invalid workflow file ... yaml syntax` | Use spaces, not tabs |
| Dockerfile not found in CI | `actions/checkout` must be the first step |
