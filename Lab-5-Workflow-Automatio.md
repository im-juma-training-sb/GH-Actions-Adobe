# Lab 9: Workflow Automation

## Overview

**Duration:** 60 minutes  
**Difficulty:** Intermediate

Learn how to automate repository housekeeping tasks like managing stale issues, auto-labeling, welcoming contributors, and creating operational workflows that reduce manual maintenance effort.

### Learning Objectives

By the end of this lab, you will:
- Automate issue and PR management workflows
- Use scheduled triggers for recurring tasks
- Implement auto-labeling based on PR content
- Welcome new contributors automatically
- Close stale issues and pull requests
- Generate automated reports

### Prerequisites

- Completed Lab 1 (Matrix Strategies)
- Completed Lab 8 (GitHub API Integration)
- Understanding of issue and PR workflows

---

## Exercise 1: Welcome New Contributors (10 minutes)

### Greet first-time contributors automatically

1. In your repository, go to **Code** → **Add file** → **Create new file**.

2. Name the file `.github/workflows/automation.yml`.

3. Add this workflow:

```yaml
name: Automation

on:
  issues:
    types: [opened]
  pull_request_target:
    types: [opened]

jobs:
  welcome:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
```

4. Commit the file.

5. Create a new issue in your repository to test the welcome message.

6. Check that the bot comments on your issue.

### What just happened?

The `actions/first-interaction` action automatically welcomes new contributors:
- Triggers on `issues: [opened]` and `pull_request_target: [opened]` events
- Uses `pull_request_target` instead of `pull_request` to safely access secrets when responding to external PRs
- `permissions:` grants `issues: write` and `pull-requests: write` so the action can post comments
- Detects if this is the contributor's first issue or PR in the repository
- Posts a customized welcome message automatically
- Improves contributor experience and sets expectations

---

## Exercise 2: Auto-Label Based on File Changes (10 minutes)

### Apply labels automatically based on PR content

1. Edit `.github/workflows/automation.yml`.

2. Add a new job for auto-labeling:

```yaml
name: Automation

on:
  issues:
    types: [opened]
  pull_request_target:
    types: [opened]

jobs:
  welcome:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
  
  auto-label:
    if: github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Label based on changed files
        uses: actions/labeler@v5
        with:
          repo-token: ${{ github.token }}
```

3. Create a new file `.github/labeler.yml` with these label rules:

```yaml
documentation:
  - changed-files:
    - any-glob-to-any-file: ['docs/**', '*.md']

workflows:
  - changed-files:
    - any-glob-to-any-file: '.github/workflows/**'

automation:
  - changed-files:
    - any-glob-to-any-file: '.github/**'
```

4. Commit both files.

5. Create labels in your repository: **documentation**, **workflows**, **automation** (go to **Issues** → **Labels** → **New label**).

6. Create a test PR that modifies a markdown file and observe the automatic labeling.

### What just happened?

The `actions/labeler` action applies labels based on changed files:
- `.github/labeler.yml` defines label rules using glob patterns
- `any-glob-to-any-file` matches if any changed file matches the pattern
- Labels are applied automatically when PRs are opened
- `permissions:` explicitly grants write access to pull requests
- Helps maintainers quickly identify PR scope and route reviews

---

## Exercise 3: Auto-Assign Issues to Team Members (10 minutes)

### Distribute new issues automatically

1. Edit `.github/workflows/automation.yml`.

2. Add a job that assigns issues based on labels:

```yaml
name: Automation

on:
  issues:
    types: [opened, labeled]
  pull_request_target:
    types: [opened]

jobs:
  welcome:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
  
  auto-label:
    if: github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Label based on changed files
        uses: actions/labeler@v5
        with:
          repo-token: ${{ github.token }}
  
  auto-assign:
    if: github.event_name == 'issues'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Assign issue based on labels
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          LABELS="${{ join(github.event.issue.labels.*.name, ',') }}"
          
          echo "Issue $ISSUE_NUMBER has labels: $LABELS"
          
          if [[ "$LABELS" == *"documentation"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to documentation team"
          elif [[ "$LABELS" == *"workflows"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to workflows team"
          else
            echo "No automatic assignment for these labels"
          fi
```

3. Commit the changes.

4. Create a new issue and add the **documentation** label to it.

5. Observe that the issue is automatically assigned.

### What just happened?

Automatic assignment routes issues to the right team members:
- Triggers on both `opened` and `labeled` events
- `join(github.event.issue.labels.*.name, ',')` creates a comma-separated list of labels
- Bash string matching checks for specific labels
- `gh issue edit --add-assignee` assigns the issue to team members
- In production, replace `${{ github.repository_owner }}` with actual team member usernames

---

## Exercise 4: Close Stale Issues (10 minutes)

### Clean up inactive issues automatically

1. Edit `.github/workflows/automation.yml`.

2. Add a scheduled job to close stale issues:

```yaml
name: Automation

on:
  issues:
    types: [opened, labeled]
  pull_request_target:
    types: [opened]
  schedule:
    - cron: '0 0 * * *'

jobs:
  welcome:
    if: github.event_name == 'issues' || github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
  
  auto-label:
    if: github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Label based on changed files
        uses: actions/labeler@v5
        with:
          repo-token: ${{ github.token }}
  
  auto-assign:
    if: github.event_name == 'issues'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Assign issue based on labels
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          LABELS="${{ join(github.event.issue.labels.*.name, ',') }}"
          
          echo "Issue $ISSUE_NUMBER has labels: $LABELS"
          
          if [[ "$LABELS" == *"documentation"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to documentation team"
          elif [[ "$LABELS" == *"workflows"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to workflows team"
          else
            echo "No automatic assignment for these labels"
          fi
  
  close-stale:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Close stale issues and PRs
        uses: actions/stale@v9
        with:
          repo-token: ${{ github.token }}
          days-before-stale: 30
          days-before-close: 7
          stale-issue-message: 'This issue has been inactive for 30 days and will close in 7 days without activity.'
          stale-pr-message: 'This PR has been inactive for 30 days and will close in 7 days without activity.'
          stale-issue-label: 'stale'
          stale-pr-label: 'stale'
          exempt-issue-labels: 'keep-open,enhancement'
          exempt-pr-labels: 'keep-open,in-progress'
```

3. Commit the changes.

4. Since scheduled workflows run daily, you'll be able to test this manually in Exercise 6 when `workflow_dispatch` is added. For now, commit and move on — the schedule will run at midnight UTC.

### What just happened?

The `actions/stale` action manages inactive issues and PRs:
- `schedule: [cron: '0 0 * * *']` runs daily at midnight UTC
- `days-before-stale: 30` marks items inactive after 30 days
- `days-before-close: 7` closes them 7 days after being marked stale
- Warning messages notify contributors before closing
- `exempt-issue-labels` protects important issues from auto-closure
- Keeps the issue tracker clean without manual intervention

---

## Exercise 5: Request Missing Information (10 minutes)

### Prompt users for required details

1. Edit `.github/workflows/automation.yml`.

2. Add a job that checks for missing information in new issues:

```yaml
name: Automation

on:
  issues:
    types: [opened, labeled]
  pull_request_target:
    types: [opened]
  schedule:
    - cron: '0 0 * * *'

jobs:
  welcome:
    if: github.event_name == 'issues' || github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
  
  auto-label:
    if: github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Label based on changed files
        uses: actions/labeler@v5
        with:
          repo-token: ${{ github.token }}
  
  auto-assign:
    if: github.event_name == 'issues' && github.event.action == 'labeled'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Assign issue based on labels
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          LABELS="${{ join(github.event.issue.labels.*.name, ',') }}"
          
          echo "Issue $ISSUE_NUMBER has labels: $LABELS"
          
          if [[ "$LABELS" == *"documentation"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to documentation team"
          elif [[ "$LABELS" == *"workflows"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to workflows team"
          else
            echo "No automatic assignment for these labels"
          fi
  
  check-issue-content:
    if: github.event_name == 'issues' && github.event.action == 'opened'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Check for required information
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          
          MISSING=()
          
          if [[ ! "$ISSUE_BODY" =~ "Steps to reproduce" ]] && [[ ! "$ISSUE_BODY" =~ "steps to reproduce" ]]; then
            MISSING+=("steps to reproduce")
          fi
          
          if [[ ! "$ISSUE_BODY" =~ "Expected behavior" ]] && [[ ! "$ISSUE_BODY" =~ "expected behavior" ]]; then
            MISSING+=("expected behavior")
          fi
          
          if [ ${#MISSING[@]} -gt 0 ]; then
            COMMENT="Thanks for the issue! It looks like some information is missing:\n\n"
            for item in "${MISSING[@]}"; do
              COMMENT+="- $item\n"
            done
            COMMENT+="\nPlease update the issue with these details so we can assist you better."
            
            gh issue comment $ISSUE_NUMBER --body "$COMMENT" --repo ${{ github.repository }}
            gh issue edit $ISSUE_NUMBER --add-label "needs-info" --repo ${{ github.repository }}
          fi
  
  close-stale:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Close stale issues and PRs
        uses: actions/stale@v9
        with:
          repo-token: ${{ github.token }}
          days-before-stale: 30
          days-before-close: 7
          stale-issue-message: 'This issue has been inactive for 30 days and will close in 7 days without activity.'
          stale-pr-message: 'This PR has been inactive for 30 days and will close in 7 days without activity.'
          stale-issue-label: 'stale'
          stale-pr-label: 'stale'
          exempt-issue-labels: 'keep-open,enhancement'
          exempt-pr-labels: 'keep-open,in-progress'
```

3. Commit the changes.

4. Create a **needs-info** label in your repository.

5. Open a new issue without the phrases "steps to reproduce" or "expected behavior" in the body.

6. Check that the bot comments requesting more information and adds the label.

### What just happened?

Content validation ensures issues have sufficient detail:
- Issue body is passed via `env:` block to prevent script injection from user-controlled input
- Bash regex `=~` checks if phrases exist in the issue body
- `MISSING=()` array tracks what information is missing
- The workflow posts a helpful comment listing missing items
- `--add-label "needs-info"` flags the issue for follow-up
- This reduces back-and-forth with contributors and speeds up triage

---

## Exercise 6: Generate Weekly Activity Report (10 minutes)

### Create automated summaries

1. Edit `.github/workflows/automation.yml`.

2. Add a job that generates weekly reports:

```yaml
name: Automation

on:
  issues:
    types: [opened, labeled]
  pull_request_target:
    types: [opened]
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  welcome:
    if: github.event_name == 'issues' || github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Welcome first-time contributor
        uses: actions/first-interaction@v1
        with:
          repo-token: ${{ github.token }}
          issue-message: |
            Thanks for opening your first issue! We'll review it soon.
            Please make sure you've provided all the necessary details.
          pr-message: |
            Thanks for your first pull request! A maintainer will review it shortly.
            Please ensure all tests pass and the PR description is complete.
  
  auto-label:
    if: github.event_name == 'pull_request_target'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Label based on changed files
        uses: actions/labeler@v5
        with:
          repo-token: ${{ github.token }}
  
  auto-assign:
    if: github.event_name == 'issues' && github.event.action == 'labeled'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Assign issue based on labels
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          LABELS="${{ join(github.event.issue.labels.*.name, ',') }}"
          
          echo "Issue $ISSUE_NUMBER has labels: $LABELS"
          
          if [[ "$LABELS" == *"documentation"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to documentation team"
          elif [[ "$LABELS" == *"workflows"* ]]; then
            gh issue edit $ISSUE_NUMBER --add-assignee "${{ github.repository_owner }}" --repo ${{ github.repository }}
            echo "Assigned to workflows team"
          else
            echo "No automatic assignment for these labels"
          fi
  
  check-issue-content:
    if: github.event_name == 'issues' && github.event.action == 'opened'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Check for required information
        env:
          GH_TOKEN: ${{ github.token }}
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          
          MISSING=()
          
          if [[ ! "$ISSUE_BODY" =~ "Steps to reproduce" ]] && [[ ! "$ISSUE_BODY" =~ "steps to reproduce" ]]; then
            MISSING+=("steps to reproduce")
          fi
          
          if [[ ! "$ISSUE_BODY" =~ "Expected behavior" ]] && [[ ! "$ISSUE_BODY" =~ "expected behavior" ]]; then
            MISSING+=("expected behavior")
          fi
          
          if [ ${#MISSING[@]} -gt 0 ]; then
            COMMENT="Thanks for the issue! It looks like some information is missing:\n\n"
            for item in "${MISSING[@]}"; do
              COMMENT+="- $item\n"
            done
            COMMENT+="\nPlease update the issue with these details so we can assist you better."
            
            gh issue comment $ISSUE_NUMBER --body "$COMMENT" --repo ${{ github.repository }}
            gh issue edit $ISSUE_NUMBER --add-label "needs-info" --repo ${{ github.repository }}
          fi
  
  close-stale:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    
    steps:
      - name: Close stale issues and PRs
        uses: actions/stale@v9
        with:
          repo-token: ${{ github.token }}
          days-before-stale: 30
          days-before-close: 7
          stale-issue-message: 'This issue has been inactive for 30 days and will close in 7 days without activity.'
          stale-pr-message: 'This PR has been inactive for 30 days and will close in 7 days without activity.'
          stale-issue-label: 'stale'
          stale-pr-label: 'stale'
          exempt-issue-labels: 'keep-open,enhancement'
          exempt-pr-labels: 'keep-open,in-progress'
  
  weekly-report:
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    permissions:
      issues: write
    
    steps:
      - name: Generate weekly activity report
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          WEEK_AGO=$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-7d +%Y-%m-%dT%H:%M:%SZ)
          
          ISSUES_CREATED=$(gh issue list --repo ${{ github.repository }} --search "created:>=$WEEK_AGO" --json number,title --jq 'length')
          ISSUES_CLOSED=$(gh issue list --state closed --repo ${{ github.repository }} --search "closed:>=$WEEK_AGO" --json number --jq 'length')
          PRS_OPENED=$(gh pr list --repo ${{ github.repository }} --search "created:>=$WEEK_AGO" --json number --jq 'length')
          PRS_MERGED=$(gh pr list --state merged --repo ${{ github.repository }} --search "merged:>=$WEEK_AGO" --json number --jq 'length')
          
          REPORT="## Weekly Activity Report
          
          **Week ending:** $(date +%Y-%m-%d)
          
          ### Issues
          - Created: $ISSUES_CREATED
          - Closed: $ISSUES_CLOSED
          
          ### Pull Requests
          - Opened: $PRS_OPENED
          - Merged: $PRS_MERGED
          
          ---
          
          Generated automatically by workflow run ${{ github.run_id }}"
          
          gh issue create \
            --title "Weekly Activity Report - $(date +%Y-%m-%d)" \
            --body "$REPORT" \
            --label "report"
```

3. Commit the changes.

4. Create a **report** label in your repository (go to **Issues** → **Labels** → **New label**).

5. Go to **Actions** → **Automation** → **Run workflow** to generate a report.

6. Check the **Issues** tab for the new report.

### What just happened?

Automated reports provide project visibility:
- `date -u -d '7 days ago'` calculates the date one week ago (with macOS fallback)
- `gh issue list --search "created:>=$WEEK_AGO"` filters items by date
- `--json number --jq 'length'` counts results
- Combines metrics into a formatted markdown report
- Schedule this job with `cron: '0 9 * * MON'` for weekly Monday reports
- Team leads get consistent updates without manual tracking

---

## What You've Learned

### Event-Driven Automation

Workflows respond to repository events:
- `issues: [opened]` triggers when issues are created
- `pull_request_target: [opened]` safely handles external PRs
- `schedule:` runs workflows on a cron schedule
- `workflow_dispatch:` allows manual triggering
- Multiple triggers in one workflow enable complex automation

### First-Party Actions

GitHub provides pre-built automation actions:
- `actions/first-interaction` welcomes new contributors
- `actions/labeler` applies labels based on file changes
- `actions/stale` closes inactive issues and PRs
- These actions reduce custom code and maintenance

### Content Analysis

Workflows can inspect and validate content:
- Access issue/PR bodies via `${{ github.event.issue.body }}`
- Use bash regex (`=~`) to check for required phrases
- Post comments requesting missing information
- Add labels to flag issues needing attention

### GitHub CLI Integration

`gh` enables powerful automation:
- `gh issue create/edit/comment` manages issues
- `gh pr list` with `--search` filters PRs by date and status
- `--json` and `jq` extract specific data for reports
- Combine with date calculations for time-based queries

### Scheduled Workflows

Cron schedules enable recurring tasks:
- `cron: '0 0 * * *'` runs daily at midnight UTC
- `cron: '0 9 * * MON'` runs weekly on Mondays at 9 AM UTC
- Use for stale issue cleanup, weekly reports, and housekeeping
- Add `workflow_dispatch` to test scheduled workflows manually

### Label-Based Routing

Labels trigger conditional logic:
- `join(github.event.issue.labels.*.name, ',')` creates label list
- Bash string matching routes issues to teams
- `gh issue edit --add-assignee` assigns automatically
- Scales team triage without manual intervention

### Real-World Applications

Automation reduces maintenance burden:
- Welcome messages improve contributor experience
- Auto-labeling speeds up PR review routing
- Stale issue cleanup keeps backlog manageable
- Content validation ensures sufficient detail
- Weekly reports provide leadership visibility
- Team members focus on high-value work instead of housekeeping
