# Lab 1: Events and Triggers

**Duration:** 45-60 minutes  
**Level:** Beginner  


---

## Overview

This lab explores the different events that can trigger GitHub Actions workflows. You'll learn how to configure workflows to respond to repository events, schedule automated tasks, and filter triggers based on branches, paths, and activity types.

No local setup required—all exercises are performed directly in GitHub.

---

## Exercise 1: Push Event with Branch Filtering (10 minutes)

### Create a workflow that only runs on specific branches

1. In your `github-actions-fundamentals` repository, navigate to **Code** tab.

2. Click **Add file** → **Create new file**.

3. Name it `.github/workflows/branch-specific.yml`.

4. Paste this content:

```yaml
name: Branch Specific Workflow

on:
  push:
    branches:
      - main
      - develop
      - 'release/**'

jobs:
  branch-info:
    runs-on: ubuntu-latest
    
    steps:
      - name: Show branch information
        run: |
          echo "Workflow triggered on branch: ${{ github.ref_name }}"
          echo "Full ref: ${{ github.ref }}"
          echo "Event: ${{ github.event_name }}"
```

5. Click **Commit changes** → commit directly to `main`.

6. Go to **Actions** tab and observe the workflow run.

7. Create a new branch to test:
   - Go to **Code** tab
   - Click the branch dropdown (shows "main")
   - Type `develop` in the text field
   - Click **Create branch: develop from main**

8. Make a change on the `develop` branch (edit README.md)

9. **Observe**: The workflow runs on `develop` as well.

10. Create a branch named `feature/test` and make a change.

11. **Observe**: The workflow does NOT run (not in the allowed branches list).

### What just happened?

The `branches:` filter specifies which branches trigger the workflow on push events:
- Exact matches: `main`, `develop`
- Wildcard patterns: `release/**` matches `release/v1`, `release/v2.0`, etc.
- Branches not listed won't trigger the workflow

---

## Exercise 2: Path Filtering (10 minutes)

### Run workflows only when specific files change

1. Create a new workflow: `.github/workflows/docs-only.yml`

```yaml
name: Documentation Updates

on:
  push:
    paths:
      - 'docs/**'
      - '**.md'
    branches:
      - main

jobs:
  docs-check:
    runs-on: ubuntu-latest
    
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: List changed files
        run: |
          echo "Documentation files were modified"
          echo "This workflow only runs when docs or markdown files change"
      
      - name: Count markdown files
        run: |
          echo "Markdown files in repository:"
          find . -name "*.md" -type f | wc -l
```

2. Commit this workflow.

3. Edit your `README.md` file (make any change).

4. **Observe**: The "Documentation Updates" workflow runs.

5. Now create a new file `src/app.js` with some content.

6. **Observe**: The "Documentation Updates" workflow does NOT run.

### Path Filtering Options

```yaml
on:
  push:
    paths:              # Run only if these paths change
      - 'src/**'
      - '**.js'
    
    paths-ignore:       # Run EXCEPT when these paths change
      - 'docs/**'
      - '**.md'
```

**Note**: You can use `paths:` OR `paths-ignore:`, but not both together.

---

## Exercise 3: Pull Request Events (10 minutes)

### Respond to pull request activity

1. Create `.github/workflows/pr-checks.yml`:

```yaml
name: Pull Request Checks

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened
    branches:
      - main

jobs:
  pr-info:
    runs-on: ubuntu-latest
    
    steps:
      - name: Display PR information
        run: |
          echo "PR #${{ github.event.pull_request.number }}"
          echo "PR Title: ${{ github.event.pull_request.title }}"
          echo "Author: ${{ github.event.pull_request.user.login }}"
          echo "Base branch: ${{ github.event.pull_request.base.ref }}"
          echo "Head branch: ${{ github.event.pull_request.head.ref }}"
          echo "Action: ${{ github.event.action }}"
  
  validation:
    runs-on: ubuntu-latest
    
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Check PR has description
        run: |
          if [ -z "${{ github.event.pull_request.body }}" ]; then
            echo "ERROR: Pull request has no description"
            exit 1
          else
            echo "✓ Pull request has description"
          fi
```

2. Commit this workflow to `main`.

3. Create a new branch and make a change:
   - Create branch `test-pr`
   - Edit README.md
   - Commit the change

4. Create a pull request:
   - Go to **Pull requests** tab
   - Click **New pull request**
   - Base: `main`, Compare: `test-pr`
   - Add a title and description
   - Click **Create pull request**

5. **Observe**: The workflow runs automatically.

6. Push another commit to the `test-pr` branch.

7. **Observe**: The workflow runs again (`synchronize` event).

### Pull Request Activity Types

| Type | When it Triggers |
|------|------------------|
| `opened` | PR is created |
| `synchronize` | New commits pushed to PR branch |
| `reopened` | Closed PR is reopened |
| `closed` | PR is closed or merged |
| `labeled` | Label added to PR |
| `assigned` | Someone assigned to PR |
| `review_requested` | Review requested |

---

## Exercise 4: Scheduled Workflows (10 minutes)

### Run workflows on a schedule using cron

1. Create `.github/workflows/scheduled.yml`:

```yaml
name: Scheduled Maintenance

on:
  schedule:
    # Runs at 00:00 UTC every day
    - cron: '0 0 * * *'
  
  workflow_dispatch:  # Allow manual trigger for testing

jobs:
  daily-task:
    runs-on: ubuntu-latest
    
    steps:
      - name: Show current time
        run: |
          echo "Scheduled task running at:"
          date -u
          echo "Next run: Tomorrow at 00:00 UTC"
      
      - name: Perform maintenance
        run: |
          echo "Running daily maintenance tasks..."
          echo "- Clear caches"
          echo "- Update dependencies"
          echo "- Generate reports"
```

2. Commit the workflow.

3. **Test manually**:
   - Go to **Actions** tab
   - Select "Scheduled Maintenance"
   - Click **Run workflow**

### Cron Syntax Guide

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

**Common Examples:**

| Cron | Description |
|------|-------------|
| `0 0 * * *` | Every day at midnight UTC |
| `0 */6 * * *` | Every 6 hours |
| `0 0 * * 1` | Every Monday at midnight |
| `0 9 * * 1-5` | Weekdays at 9 AM |
| `0 0 1 * *` | First day of every month |

**Note**: Scheduled workflows may have a delay of up to 15 minutes during high load.

---

## Exercise 5: Manual Trigger with Inputs (10 minutes)

### Create workflows with user inputs

1. Create `.github/workflows/deploy.yml`:

```yaml
name: Manual Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      
      version:
        description: 'Version to deploy'
        required: true
        default: 'latest'
      
      dry-run:
        description: 'Perform dry run only'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Display deployment configuration
        run: |
          echo "=== Deployment Configuration ==="
          echo "Environment: ${{ github.event.inputs.environment }}"
          echo "Version: ${{ github.event.inputs.version }}"
          echo "Dry Run: ${{ github.event.inputs.dry-run }}"
          echo "Triggered by: ${{ github.actor }}"
      
      - name: Validate environment
        run: |
          if [ "${{ github.event.inputs.environment }}" = "production" ]; then
            echo "⚠️  WARNING: Deploying to PRODUCTION"
            echo "Requires additional approval..."
          fi
      
      - name: Execute deployment
        if: github.event.inputs.dry-run != 'true'
        run: |
          echo "🚀 Deploying version ${{ github.event.inputs.version }}"
          echo "   to ${{ github.event.inputs.environment }}"
          echo "Deployment completed successfully!"
      
      - name: Dry run mode
        if: github.event.inputs.dry-run == 'true'
        run: |
          echo "📋 DRY RUN MODE"
          echo "No actual deployment performed"
          echo "Validation checks passed"
```

2. Commit the workflow.

3. **Test the inputs**:
   - Go to **Actions** tab
   - Click "Manual Deployment"
   - Click **Run workflow**
   - Select different options from the dropdowns
   - Try checking/unchecking the dry-run option
   - Click **Run workflow**

4. Try different combinations and observe how the workflow behaves.

### Input Types

| Type | Description | Example |
|------|-------------|---------|
| `choice` | Dropdown selection | environment selection |
| `boolean` | Checkbox | dry-run toggle |
| `string` | Text input | version number |
| `environment` | Environment selector | deployment target |

---

## Exercise 6: Multiple Event Types (10 minutes)

### Combine different triggers in one workflow

1. Create `.github/workflows/multi-trigger.yml`:

```yaml
name: Multi-Trigger Workflow

on:
  push:
    branches: [main]
  
  pull_request:
    branches: [main]
  
  schedule:
    - cron: '0 0 * * 1'  # Weekly on Monday
  
  workflow_dispatch:
    inputs:
      reason:
        description: 'Reason for manual trigger'
        required: false

jobs:
  identify-trigger:
    runs-on: ubuntu-latest
    
    steps:
      - name: Identify how workflow was triggered
        run: |
          echo "Event: ${{ github.event_name }}"
          
          if [ "${{ github.event_name }}" = "push" ]; then
            echo "✓ Triggered by push to ${{ github.ref_name }}"
            echo "Commit: ${{ github.sha }}"
          
          elif [ "${{ github.event_name }}" = "pull_request" ]; then
            echo "✓ Triggered by pull request #${{ github.event.pull_request.number }}"
            echo "PR: ${{ github.event.pull_request.title }}"
          
          elif [ "${{ github.event_name }}" = "schedule" ]; then
            echo "✓ Triggered by scheduled cron job"
            echo "Weekly maintenance run"
          
          elif [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "✓ Triggered manually by ${{ github.actor }}"
            echo "Reason: ${{ github.event.inputs.reason }}"
          fi
      
      - name: Conditional step for push only
        if: github.event_name == 'push'
        run: echo "This only runs on push events"
      
      - name: Conditional step for PR only
        if: github.event_name == 'pull_request'
        run: echo "This only runs on pull request events"
```

2. Commit the workflow.

3. **Test different triggers**:
   - Push to main (already triggered by commit)
   - Create a PR
   - Run manually with a reason
   - Observe how each trigger behaves differently

---

## Exercise 7: Advanced Filtering (10 minutes)

### Combine branch, path, and tag filters

1. Create `.github/workflows/advanced-filters.yml`:

```yaml
name: Advanced Filtering

on:
  push:
    branches:
      - main
      - 'release/**'
    tags:
      - 'v*'
    paths:
      - 'src/**'
      - 'package.json'
    paths-ignore:
      - 'src/tests/**'

jobs:
  filtered-build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: <your-org>/<your-checkout-action>@<version>  # Replace with your internal checkout action
      
      - name: Show trigger details
        run: |
          echo "Ref: ${{ github.ref }}"
          echo "Ref name: ${{ github.ref_name }}"
          echo "Ref type: ${{ github.ref_type }}"
          
          if [ "${{ github.ref_type }}" = "tag" ]; then
            echo "✓ Triggered by tag: ${{ github.ref_name }}"
          else
            echo "✓ Triggered by branch: ${{ github.ref_name }}"
          fi
      
      - name: Build application
        run: |
          echo "Building application..."
          echo "Only runs when src/ or package.json changes"
          echo "Does not run for test file changes"
```

2. Commit the workflow.

3. **Test the filters**:
   - Edit a file in `src/` (if it doesn't exist, create `src/app.js`)
   - Observe the workflow runs
   - Edit only test files or documentation
   - Observe the workflow does NOT run

### Filter Patterns

```yaml
branches:
  - main                # Exact match
  - 'release/**'        # Wildcard: release/v1, release/v2.0
  - '!experimental'     # Exclusion: all except experimental

tags:
  - 'v*'                # v1, v2.0, v3.1.0
  - 'release-*'         # release-1, release-2

paths:
  - '**.js'             # Any .js file anywhere
  - 'src/**'            # Anything under src/
  - '!**/README.md'     # Exclude README files
```

---

## What You've Learned

Through these exercises, you've mastered workflow triggers:

### 1. **Push Events** (Exercise 1)
- Filter by branches using exact names or wildcards
- Use patterns like `release/**` for flexible matching
- Exclude branches with `!branch-name`

### 2. **Path Filtering** (Exercise 2)
- Trigger only when specific files/directories change
- Use `paths:` for inclusion, `paths-ignore:` for exclusion
- Combine with branch filters for precise control

### 3. **Pull Request Events** (Exercise 3)
- Respond to different PR activities (opened, synchronize, etc.)
- Access PR metadata in workflows
- Validate PR requirements automatically

### 4. **Scheduled Workflows** (Exercise 4)
- Use cron syntax for time-based automation
- Schedule maintenance, reports, or cleanup tasks
- Always add `workflow_dispatch` for testing

### 5. **Manual Triggers** (Exercise 5)
- Create interactive workflows with user inputs
- Support different input types (choice, boolean, string)
- Build deployment and operational workflows

### 6. **Multiple Triggers** (Exercise 6)
- Combine different event types in one workflow
- Use conditionals to run steps based on trigger type
- Create flexible, multi-purpose workflows

### 7. **Advanced Filtering** (Exercise 7)
- Combine branches, tags, and paths
- Use wildcards and exclusions
- Optimize workflows to run only when needed

---

## Best Practices

**DO:**
- Use `workflow_dispatch` for testing scheduled workflows
- Combine filters to reduce unnecessary runs
- Add descriptive names to workflow triggers
- Use `paths-ignore` for documentation changes
- Test filters on feature branches before main

**DON'T:**
- Over-filter and miss important changes
- Use both `paths` and `paths-ignore` together
- Forget timezones (cron uses UTC)
- Create overly complex filter combinations
- Skip testing manual trigger inputs

---

## Common Event Types Reference

```yaml
on:
  push:                   # Code pushed
  pull_request:           # PR opened/updated
  pull_request_target:    # PR from fork (use carefully)
  issues:                 # Issue activity
  issue_comment:          # Comment on issue/PR
  release:                # Release published
  schedule:               # Cron schedule
  workflow_dispatch:      # Manual trigger
  repository_dispatch:    # External webhook
  create:                 # Branch/tag created
  delete:                 # Branch/tag deleted
  fork:                   # Repository forked
  watch:                  # Repository starred
```

---


