# GitHub Actions and GitLab CI Quick Reference Guide

## GitHub Actions CLI
```bash
gh auth status
gh workflow list
gh workflow view <workflow>
gh workflow run <workflow>
gh run list
gh run view <run-id>
gh run view <run-id> --log
gh run watch <run-id>
gh run rerun <run-id>
gh run cancel <run-id>
```

## GitHub Actions Workflow Basics
```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

## GitHub Actions Common Features
```yaml
permissions:
  contents: read
  id-token: write

strategy:
  matrix:
    node: [20, 22]

env:
  NODE_ENV: test
```

## GitHub Actions Cache and Artifacts
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}

- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/

- uses: actions/download-artifact@v4
  with:
    name: test-results
```

## GitLab CI Basics
```yaml
stages:
  - test
  - build

test:
  stage: test
  image: node:22
  script:
    - npm ci
    - npm test
```

## GitLab CI Rules and Artifacts
```yaml
job:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
  artifacts:
    when: always
    paths:
      - test-results/
```

## GitLab CLI
```bash
glab auth status
glab pipeline list
glab pipeline view <id>
glab pipeline ci lint
glab ci view
glab mr list
glab mr view <id>
```

## Common Debugging
```bash
git rev-parse HEAD
git status
git diff --name-only origin/main...HEAD
printenv | sort
cat .github/workflows/<workflow>.yml
cat .gitlab-ci.yml
```
