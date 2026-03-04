# Lab 1: Matrix Strategies

**Trainer:** GitHub Actions Intermediate Training for Enterprise  
**Duration:** 45-60 minutes  
**Prerequisites:** Basic GitHub Actions knowledge, access to a GitHub repository

---

## Overview

This lab demonstrates how to scale a single job across many OS and runtime combinations without wasting runner minutes. You'll create a matrix workflow from scratch, then incrementally modify it to understand how `fail-fast`, `continue-on-error`, `max-parallel`, `include`, and `exclude` change execution behavior.

No local setup required—the workflow simply prints status messages. You just need a GitHub account with Actions enabled.

---

## Step-by-Step

### Create your repository

1. In GitHub, create a new repository (or use an existing one).

2. Make sure Actions are enabled (Settings → Actions → General → Allow actions from your organization).

---

## Exercise 1: Create the Basic Matrix (10 minutes)

### Build your first matrix workflow

1. In your GitHub repository, navigate to the **Code** tab.

2. Click **Add file** → **Create new file**.

3. Name the file `.github/workflows/matrix.yml` (GitHub will create the folders automatically).

4. Paste this initial content:

```yaml
name: Matrix Mastery

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
    
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Display configuration
        run: |
          echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
          echo "Runner OS: $(uname -a 2>/dev/null || ver)"
          echo "Matrix combination verified successfully"
```

5. Click **Commit changes...** → **Commit changes** (commit directly to main).

6. Go to **Actions** tab → **Matrix Mastery** → **Run workflow** → **Run workflow**.

7. **Observe**: You should see 4 jobs running (2 OS × 2 Node versions), each printing its configuration details.

### What just happened?

The matrix automatically expands into 4 separate jobs:
- `ubuntu-latest` + Node 18
- `ubuntu-latest` + Node 20
- `windows-latest` + Node 18
- `windows-latest` + Node 20

Each job runs independently and prints its configuration. The matrix values are accessible via `${{ matrix.os }}` and `${{ matrix.node }}`.

---

## Exercise 2: Add Parallelism Control (10 minutes)

### Limit concurrent jobs

1. Go to your repository's **Code** tab.

2. Click on `.github/workflows/matrix.yml` to open it.

3. Click the **pencil icon** (Edit this file).

4. Modify the `strategy:` section to add `max-parallel`:

```yaml
    strategy:
      max-parallel: 2
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
```

5. Click **Commit changes...** → **Commit changes**.

6. Go to **Actions** → run the workflow again.

7. **Watch the grid**: Only 2 jobs run at the same time now, even though there are 4 total jobs.

### What just happened?

`max-parallel: 2` limits how many matrix jobs run simultaneously. This is useful when:
- You have limited runner capacity
- You want to control resource usage
- You need to avoid rate limits on external services

The 4 jobs still all run, but they queue up—only 2 at a time.

---

## Exercise 3: Add an Experimental Configuration (10 minutes)

### Use `include` to add a special matrix combination

1. Edit `.github/workflows/matrix.yml` again.

2. Add an `include:` section under your matrix:

```yaml
    strategy:
      max-parallel: 2
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
        include:
          - os: ubuntu-latest
            node: 22
            experimental: true
```

3. Add a step to mark experimental jobs:

```yaml
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Display configuration
        run: |
          echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
          echo "Runner OS: $(uname -a 2>/dev/null || ver)"
          echo "Matrix combination verified successfully"
      
      - name: Mark as experimental
        if: matrix.experimental == true
        run: echo "WARNING: This is an EXPERIMENTAL configuration"
```

4. Commit the changes.

5. Run the workflow and **observe**: You now have 5 jobs (4 regular + 1 experimental).

### What just happened?

`include` adds extra combinations to your matrix beyond the base permutations. You can also add custom properties (like `experimental: true`) that are only available on that specific combination.

The experimental job has access to `matrix.experimental`, which the other 4 jobs don't have.

---

## Exercise 4: Make Experimental Jobs Non-Blocking (10 minutes)

### Use `continue-on-error` for experimental configurations

1. Edit `.github/workflows/matrix.yml` again.

2. Add `continue-on-error` at the job level, using the experimental flag:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental || false }}
    
    strategy:
      max-parallel: 2
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
        include:
          - os: ubuntu-latest
            node: 22
            experimental: true
```

3. Now let's simulate a failure. Add a failing step for the experimental job:

```yaml
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action

      - name: Display configuration
        run: echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
      
      - name: Mark as experimental
        if: matrix.experimental == true
        run: echo "WARNING: This is an EXPERIMENTAL configuration"
      
      - name: Intentional failure for experimental
        if: matrix.experimental == true
        run: exit 1
      
      - name: Verify configuration
        run: |
          echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
          echo "Matrix combination verified successfully"
```

4. Commit and run the workflow.

5. **Watch**: The experimental job fails (shows red), but the workflow overall still passes (green check).

### What just happened?

`continue-on-error: ${{ matrix.experimental || false }}` means:
- If `matrix.experimental` is `true`, the job can fail without failing the workflow
- If `matrix.experimental` is `false` (or doesn't exist), job failures cause workflow failure

This is perfect for testing bleeding-edge versions without blocking your main pipeline.

---

## Exercise 5: Remove the Failure and Exclude a Combo (10 minutes)

### Clean up and use `exclude`

1. Edit `.github/workflows/matrix.yml` again.

2. First, remove the intentional failure step (delete it completely):

```yaml
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Display configuration
        run: |
          echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
          echo "Runner OS: $(uname -a 2>/dev/null || ver)"
          echo "Matrix combination verified successfully"
      
      - name: Mark as experimental
        if: matrix.experimental == true
        run: echo "WARNING: This is an EXPERIMENTAL configuration"
```

3. Now add an `exclude:` section to your strategy (remove ubuntu + Node 20):

```yaml
    strategy:
      max-parallel: 2
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
        exclude:
          - os: ubuntu-latest
            node: 20
        include:
          - os: ubuntu-latest
            node: 22
            experimental: true
```

4. Commit and run the workflow.

5. **Count**: You now have 4 jobs instead of 5:
   - `ubuntu` + Node 18 (included)
   - `ubuntu` + Node 20 (excluded)
   - `ubuntu` + Node 22 (experimental)
   - `windows` + Node 18 (included)
   - `windows` + Node 20 (included)

### What just happened?

`exclude` removes specific combinations from the matrix. This is useful when:
- Certain OS/version combos don't make sense
- You want to save runner minutes by skipping redundant tests
- A particular combination is deprecated

`exclude` runs before `include`, so you can exclude base combos and then add back specific configurations.

---

## Exercise 6: Test Fail-Fast Behavior (10 minutes)

### Force a failure and watch cancellation

1. Edit `.github/workflows/matrix.yml` one more time.

2. Add `fail-fast: true` to the strategy section:

```yaml
    strategy:
      fail-fast: true
      max-parallel: 2
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]
        exclude:
          - os: ubuntu-latest
            node: 20
        include:
          - os: ubuntu-latest
            node: 22
            experimental: true
```

3. Add a step that always fails at the top of your steps:

```yaml
    steps:
      - name: Intentional failure
        run: exit 1
      
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Display configuration
        run: |
          echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
          echo "Runner OS: $(uname -a 2>/dev/null || ver)"
          echo "Matrix combination verified successfully"
      
      - name: Mark as experimental
        if: matrix.experimental == true
        run: echo "WARNING: This is an EXPERIMENTAL configuration"
```

4. Commit and run the workflow.

5. **Watch carefully**: One job fails immediately, and the other jobs get cancelled (except the experimental one, because of `continue-on-error`).

### What just happened?

`fail-fast: true` (the default) means when any job fails, GitHub Actions immediately cancels all other running and queued jobs. This saves runner minutes when you know the whole build is broken.

`fail-fast: false` would let all jobs run to completion regardless of failures, useful when you want to see all test results.

6. **Clean up**: Delete the "Intentional failure" step and commit again to restore the working workflow.

---

## What You've Learned

By incrementally building and modifying a single workflow file, you've seen how matrix strategies:

### 1. **Basic Matrix** (Exercise 1)
- Automatically expand a job across multiple OS and runtime combinations
- Reference matrix values using `${{ matrix.* }}`
- Generate NxM job permutations from N and M axis values

### 2. **Parallelism Control** (Exercise 2)
- Use `max-parallel` to limit concurrent jobs
- Balance speed vs. resource constraints
- Queue jobs that exceed the parallel limit

### 3. **Include for Special Combos** (Exercise 3)
- Add extra matrix combinations beyond base permutations
- Attach custom properties to specific combinations
- Access those properties with conditionals

### 4. **Continue-on-Error** (Exercise 4)
- Allow experimental configurations to fail without blocking the pipeline
- Use conditional expressions: `${{ matrix.experimental || false }}`
- Mark jobs as "allowed to fail" dynamically

### 5. **Exclude to Skip Combos** (Exercise 5)
- Remove specific unwanted combinations from the matrix
- Save runner minutes by skipping redundant or incompatible tests
- `exclude` runs before `include`

### 6. **Fail-Fast Behavior** (Exercise 6)
- `fail-fast: true` cancels remaining jobs when any job fails (saves time/money)
- `fail-fast: false` lets all jobs complete (useful for comprehensive test results)
- Jobs with `continue-on-error` don't trigger fail-fast cancellation

---

## Best Practices

**DO:**
- Start with a simple matrix, then add complexity incrementally
- Use `max-parallel` when you have limited runners or rate limits
- Mark experimental versions with `continue-on-error: true`
- Use `exclude` to skip invalid or expensive combinations
- Use `include` to add one-off special configurations

**DON'T:**
- Create overly complex matrices that are hard to debug
- Forget that `fail-fast: true` is the default
- Mix matrix axes carelessly (e.g., Windows + PostgreSQL may not work)
- Ignore runner costs—matrices multiply quickly!

---

## Next Steps

Now that you understand matrix strategies, you can:

1. **Apply this to real projects**: Test your app across multiple Node/Python/Ruby versions
2. **Combine with other labs**: Use matrices with job dependencies (Lab 3) and conditionals (Lab 4)
3. **Optimize costs**: Design matrices that give good coverage without redundant tests

---

## Quick Reference

```yaml
strategy:
  fail-fast: true|false       # Cancel others on first failure?
  max-parallel: N             # Run at most N jobs at once
  matrix:
    os: [ubuntu, windows]     # Base axis
    node: [18, 20]            # Base axis
    exclude:                  # Remove specific combos
      - os: ubuntu
        node: 20
    include:                  # Add special combos
      - os: ubuntu
        node: 22
        experimental: true    # Custom property
```

---

**End of Lab 1 - Matrix Strategies**

Continue to [Lab 2: Concurrency Control](./lab-02-concurrency-control.md)
