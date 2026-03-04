# Lab 5: JavaScript Actions

**Trainer:** GitHub Actions Intermediate Training for Enterprise  
**Duration:** 60-90 minutes  
**Prerequisites:** Basic JavaScript/Node.js knowledge, understanding of GitHub Actions

---

## Learning Objectives

By the end of this lab, you will be able to:
- Create custom JavaScript actions from scratch
- Define inputs and outputs for actions
- Use the GitHub Actions toolkit
- Handle errors and logging in actions
- Publish and use custom actions in workflows
- Make API calls within actions

---

## Exercise 1: Action with GitHub API Integration (25 minutes)

### Scenario
Create an action that fetches information about the repository and creates a comment on issues.

### Task
Build an action that interacts with the GitHub API.

### Steps

1. Click **Add file** → **Create new file**, name it `.github/actions/repo-info/action.yml`.

2. Paste this content:

```yaml
name: 'Repository Info'
description: 'Get repository information and statistics'
author: 'Training Team'

inputs:
  github-token:
    description: 'GitHub token for API access'
    required: true
  include-collaborators:
    description: 'Include collaborator count'
    required: false
    default: 'false'

outputs:
  repo-name:
    description: 'Repository name'
  stars:
    description: 'Number of stars'
  open-issues:
    description: 'Number of open issues'

runs:
  using: 'node20'
  main: 'index.js'
```

3. Click **Commit changes** → **Commit changes**.

4. Create `package.json`: Click **Add file** → **Create new file**, name it `.github/actions/repo-info/package.json`:

```json
{
  "name": "repo-info-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/github": "^6.0.0"
  }
}
```

5. Click **Commit changes** → **Commit changes**.

6. Create `index.js`: Click **Add file** → **Create new file**, name it `.github/actions/repo-info/index.js`:

```javascript
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  try {
    // Get inputs
    const token = core.getInput('github-token', { required: true });
    const includeCollaborators = core.getInput('include-collaborators') === 'true';
    
    // Create Octokit instance
    const octokit = github.getOctokit(token);
    
    // Get repository information
    const { owner, repo } = github.context.repo;
    
    core.info(`Fetching info for ${owner}/${repo}`);
    
    const { data: repoData } = await octokit.rest.repos.get({
      owner,
      repo
    });
    
    // Set outputs
    core.setOutput('repo-name', repoData.full_name);
    core.setOutput('stars', repoData.stargazers_count);
    core.setOutput('open-issues', repoData.open_issues_count);
    
    // Log information
    core.info(`Stars: ${repoData.stargazers_count}`);
    core.info(`Open Issues: ${repoData.open_issues_count}`);
    core.info(`Watchers: ${repoData.watchers_count}`);
    
    if (includeCollaborators) {
      const { data: collaborators } = await octokit.rest.repos.listCollaborators({
        owner,
        repo
      });
      core.info(`Collaborators: ${collaborators.length}`);
    }
    
    // Create summary
    core.summary
      .addHeading('Repository Information')
      .addTable([
        [{data: 'Metric', header: true}, {data: 'Value', header: true}],
        ['Repository', repoData.full_name],
        ['Stars', repoData.stargazers_count.toString()],
        ['Open Issues', repoData.open_issues_count.toString()],
        ['Forks', repoData.forks_count.toString()]
      ])
      .write();
    
  } catch (error) {
    core.setFailed(`Action failed with error: ${error.message}`);
  }
}

run();
```

7. Click **Commit changes** → **Commit changes**.

8. Create test workflow: Click **Add file** → **Create new file**, name it `.github/workflows/test-repo-info.yml`:

```yaml
name: Test Repo Info Action

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  contents: read
  actions: read

jobs:
  get-info:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get repository info
        id: repo-info
        uses: ./.github/actions/repo-info
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          include-collaborators: 'false'
      
      - name: Display info
        run: |
          echo "Repository: ${{ steps.repo-info.outputs.repo-name }}"
          echo "Stars: ${{ steps.repo-info.outputs.stars }}"
          echo "Open Issues: ${{ steps.repo-info.outputs.open-issues }}"
```

9. Click **Commit changes** → **Commit changes**.

10. Go to **Actions** tab → **Test Repo Info Action** → **Run workflow** → **Run workflow**.

11. View the workflow run to see the repository information.

### What Just Happened?

You built an action that interacts with the GitHub API:
- GitHub API authentication uses tokens passed as inputs: `github.getOctokit(token)` creates an authenticated client
- The `GITHUB_TOKEN` secret provides automatic authentication with appropriate repository permissions
- `github.getOctokit(token)` wraps the Octokit REST client internally, providing typed methods for all GitHub API endpoints
- `core.summary` creates rich markdown summaries displayed at the bottom of job logs using methods like `.addHeading()`, `.addTable()`, `.write()`
- `github.context` provides workflow context including repository owner, name, event data, and more

This pattern enables actions to read repository data, create issues, manage PRs, and automate GitHub operations.

### Challenge
Extend the action to also fetch the number of pull requests and add it to the outputs.

---

## Exercise 2: Action with Input Validation (20 minutes)

### Scenario
Create an action that validates and processes user inputs with proper error handling.

### Task
Build an action with comprehensive input validation.

### Steps

1. Click **Add file** → **Create new file**, name it `.github/actions/validate-inputs/action.yml`.

2. Paste this content:

```yaml
name: 'Validate Inputs'
description: 'Validate and process various input types'
author: 'Training Team'

inputs:
  environment:
    description: 'Deployment environment'
    required: true
  version:
    description: 'Version number (semver format)'
    required: true
  port:
    description: 'Port number (1-65535)'
    required: false
    default: '8080'
  enable-ssl:
    description: 'Enable SSL (true/false)'
    required: false
    default: 'true'

outputs:
  validated:
    description: 'Whether all inputs are valid'
  config:
    description: 'JSON configuration object'

runs:
  using: 'node20'
  main: 'index.js'
```

3. Click **Commit changes** → **Commit changes**.

4. Create `package.json`: Click **Add file** → **Create new file**, name it `.github/actions/validate-inputs/package.json`:

```json
{
  "name": "validate-inputs-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "semver": "^7.5.4"
  }
}
```

5. Click **Commit changes** → **Commit changes**.

6. Create `index.js`: Click **Add file** → **Create new file**, name it `.github/actions/validate-inputs/index.js`:

```javascript
const core = require('@actions/core');
const semver = require('semver');

async function run() {
  try {
    // Get inputs
    const environment = core.getInput('environment', { required: true });
    const version = core.getInput('version', { required: true });
    const port = core.getInput('port');
    const enableSSL = core.getInput('enable-ssl');
    
    // Validation flags
    let isValid = true;
    const errors = [];
    
    // Validate environment
    const validEnvironments = ['development', 'staging', 'production'];
    if (!validEnvironments.includes(environment)) {
      errors.push(`Invalid environment: ${environment}. Must be one of: ${validEnvironments.join(', ')}`);
      isValid = false;
    } else {
      core.info(`Environment: ${environment}`);
    }
    
    // Validate version (semver)
    if (!semver.valid(version)) {
      errors.push(`Invalid version: ${version}. Must be valid semver (e.g., 1.0.0)`);
      isValid = false;
    } else {
      core.info(`Version: ${version}`);
    }
    
    // Validate port
    const portNumber = parseInt(port, 10);
    if (isNaN(portNumber) || portNumber < 1 || portNumber > 65535) {
      errors.push(`Invalid port: ${port}. Must be between 1 and 65535`);
      isValid = false;
    } else {
      core.info(`Port: ${portNumber}`);
    }
    
    // Validate boolean
    if (!['true', 'false'].includes(enableSSL.toLowerCase())) {
      errors.push(`Invalid enable-ssl: ${enableSSL}. Must be true or false`);
      isValid = false;
    } else {
      core.info(`SSL Enabled: ${enableSSL}`);
    }
    
    // Report errors if any
    if (!isValid) {
      core.error('Validation failed with the following errors:');
      errors.forEach(error => core.error(`  - ${error}`));
      core.setFailed('Input validation failed');
      return;
    }
    
    // Create configuration object
    const config = {
      environment,
      version,
      port: portNumber,
      ssl: enableSSL === 'true',
      timestamp: new Date().toISOString()
    };
    
    // Set outputs
    core.setOutput('validated', 'true');
    core.setOutput('config', JSON.stringify(config));
    
    // Create summary
    core.summary
      .addHeading('Validation Successful')
      .addCodeBlock(JSON.stringify(config, null, 2), 'json')
      .write();
    
    core.info('All inputs validated successfully!');
    
  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}

run();
```

7. Click **Commit changes** → **Commit changes**.

8. Create test workflow: Click **Add file** → **Create new file**, name it `.github/workflows/test-validation.yml`:

```yaml
name: Test Input Validation

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      version:
        description: 'Version (semver)'
        required: true
        default: '1.0.0'

jobs:
  validate:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate inputs
        id: validate
        uses: ./.github/actions/validate-inputs
        with:
          environment: ${{ inputs.environment }}
          version: ${{ inputs.version }}
          port: '3000'
          enable-ssl: 'true'
      
      - name: Show configuration
        if: steps.validate.outputs.validated == 'true'
        run: |
          echo "Configuration:"
          echo '${{ steps.validate.outputs.config }}' | jq .
```

9. Click **Commit changes** → **Commit changes**.

10. Go to **Actions** tab → **Test Input Validation** → **Run workflow**, provide inputs, and click **Run workflow**.

11. View the results to see validation in action.

### What Just Happened?

You implemented comprehensive input validation:
- `core.error()` logs error messages but doesn't stop execution - useful for collecting multiple validation errors
- `core.setFailed(message)` logs an error AND marks the action as failed, stopping subsequent steps unless they use `if: always()`
- Validation should check types, ranges, allowed values, and formats before processing
- The `semver` library provides robust semantic version validation and comparison
- Structured outputs use `JSON.stringify()` to pass complex data between steps

This pattern ensures actions fail fast with clear error messages rather than producing incorrect results.

### Test Scenarios
1. Run with valid inputs
2. Run with invalid version (e.g., "1.0")
3. Run with invalid environment (e.g., "test")

---

## Exercise 3: Action with External API Calls (25 minutes)

### Scenario
Create an action that calls an external API (e.g., weather API, joke API) and processes the response.

### Task
Build an action that integrates with external services.

### Steps

1. Click **Add file** → **Create new file**, name it `.github/actions/external-api/action.yml`.

2. Paste this content:

```yaml
name: 'External API Integration'
description: 'Fetch data from external APIs'
author: 'Training Team'

inputs:
  api-type:
    description: 'Type of API to call (joke, quote, fact)'
    required: false
    default: 'joke'
  category:
    description: 'Category for the API request'
    required: false
    default: 'programming'

outputs:
  response:
    description: 'API response data'
  status:
    description: 'HTTP status code'

runs:
  using: 'node20'
  main: 'index.js'
```

3. Click **Commit changes** → **Commit changes**.

4. Create `package.json`: Click **Add file** → **Create new file**, name it `.github/actions/external-api/package.json`:

```json
{
  "name": "external-api-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/http-client": "^2.2.0"
  }
}
```

5. Click **Commit changes** → **Commit changes**.

6. Create `index.js`: Click **Add file** → **Create new file**, name it `.github/actions/external-api/index.js`:

```javascript
const core = require('@actions/core');
const { HttpClient } = require('@actions/http-client');

async function run() {
  try {
    const apiType = core.getInput('api-type');
    const category = core.getInput('category');
    
    const http = new HttpClient('github-actions-training');
    
    let url;
    let responseData;
    
    switch (apiType) {
      case 'joke':
        url = `https://official-joke-api.appspot.com/jokes/${category}/random`;
        core.info(`Fetching ${category} joke from: ${url}`);
        break;
        
      case 'quote':
        url = 'https://dummyjson.com/quotes/random';
        core.info(`Fetching random quote from: ${url}`);
        break;
        
      case 'fact':
        url = 'https://uselessfacts.jsph.pl/api/v2/facts/random?language=en';
        core.info(`Fetching random fact from: ${url}`);
        break;
        
      default:
        throw new Error(`Unknown API type: ${apiType}`);
    }
    
    // Make API request
    const response = await http.get(url);
    const statusCode = response.message.statusCode;
    
    core.info(`HTTP Status: ${statusCode}`);
    
    if (statusCode !== 200) {
      throw new Error(`API request failed with status ${statusCode}`);
    }
    
    // Parse response body
    const body = await response.readBody();
    const data = JSON.parse(body);
    
    // Process based on API type
    let output;
    
    if (apiType === 'joke') {
      const joke = data[0] || data;
      output = {
        type: 'joke',
        setup: joke.setup,
        punchline: joke.punchline
      };
      
      core.info('Joke received!');
      core.info(`Setup: ${joke.setup}`);
      core.info(`Punchline: ${joke.punchline}`);
      
    } else if (apiType === 'quote') {
      output = {
        type: 'quote',
        content: data.quote,
        author: data.author
      };
      
      core.info('Quote received!');
      core.info(`"${data.quote}" - ${data.author}`);
      
    } else if (apiType === 'fact') {
      output = {
        type: 'fact',
        text: data.text
      };
      
      core.info('Fact received!');
      core.info(data.text);
    }
    
    // Set outputs
    core.setOutput('response', JSON.stringify(output));
    core.setOutput('status', statusCode.toString());
    
    // Create summary
    core.summary
      .addHeading(`${apiType.charAt(0).toUpperCase() + apiType.slice(1)} of the Day`)
      .addQuote(output.content || output.text || `${output.setup}\n\n${output.punchline}`)
      .write();
    
  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}

run();
```

7. Click **Commit changes** → **Commit changes**.

8. Create test workflow: Click **Add file** → **Create new file**, name it `.github/workflows/test-api.yml`:

```yaml
name: Test External API Action

on:
  workflow_dispatch:
    inputs:
      api-type:
        description: 'API Type'
        type: choice
        options:
          - joke
          - quote
          - fact
        default: 'joke'
  schedule:
    - cron: '0 9 * * *'  # Daily at 9 AM UTC

jobs:
  fetch-data:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Fetch from external API
        id: api
        uses: ./.github/actions/external-api
        with:
          api-type: ${{ inputs.api-type || 'joke' }}
      
      - name: Display response
        run: |
          echo "Status: ${{ steps.api.outputs.status }}"
          echo "Response:"
          echo '${{ steps.api.outputs.response }}' | jq .
```

9. Click **Commit changes** → **Commit changes**.

10. Go to **Actions** tab → **Test External API Action** → **Run workflow**, select an API type, and click **Run workflow**.

11. View the workflow run to see the API response.

### What Just Happened?

You integrated external APIs into a custom action:
- `@actions/http-client` provides a built-in HTTP client optimized for GitHub Actions with proper user-agent headers
- Always check HTTP status codes before processing responses - non-200 codes should throw errors
- Parse JSON responses with `JSON.parse()` and handle potential parsing errors
- Use `try-catch` blocks to handle network failures, timeouts, and invalid responses gracefully
- Different APIs have different response structures - switch statements help handle multiple API types

Best practices: validate responses, implement timeouts, don't expose API keys in logs, handle rate limits.

### Challenge
Add error handling for network failures and implement retry logic.

---

## Exercise 4: Action with File Operations (20 minutes)

### Scenario
Create an action that reads files, processes content, and writes results back.

### Task
Build an action that performs file system operations.

### Steps

1. Click **Add file** → **Create new file**, name it `.github/actions/file-processor/action.yml`.

2. Paste this content:

```yaml
name: 'File Processor'
description: 'Process files and generate reports'
author: 'Training Team'

inputs:
  input-file:
    description: 'Input file path'
    required: true
  output-file:
    description: 'Output file path'
    required: true
  operation:
    description: 'Operation to perform (count-lines, uppercase, stats)'
    required: false
    default: 'stats'

outputs:
  lines-processed:
    description: 'Number of lines processed'
  output-path:
    description: 'Path to output file'

runs:
  using: 'node20'
  main: 'index.js'
```

3. Click **Commit changes** → **Commit changes**.

4. Create `package.json`: Click **Add file** → **Create new file**, name it `.github/actions/file-processor/package.json`:

```json
{
  "name": "file-processor-action",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/io": "^1.1.3"
  }
}
```

5. Click **Commit changes** → **Commit changes**.

6. Create `index.js`: Click **Add file** → **Create new file**, name it `.github/actions/file-processor/index.js`:

```javascript
const core = require('@actions/core');
const io = require('@actions/io');
const fs = require('fs').promises;
const path = require('path');

async function run() {
  try {
    const inputFile = core.getInput('input-file', { required: true });
    const outputFile = core.getInput('output-file', { required: true });
    const operation = core.getInput('operation');
    
    core.info(`Processing file: ${inputFile}`);
    core.info(`Operation: ${operation}`);
    
    // Check if input file exists
    try {
      await fs.access(inputFile);
    } catch (error) {
      throw new Error(`Input file not found: ${inputFile}`);
    }
    
    // Read input file
    const content = await fs.readFile(inputFile, 'utf8');
    const lines = content.split('\n');
    
    let result;
    let stats = {};
    
    // Perform operation
    switch (operation) {
      case 'count-lines':
        stats.totalLines = lines.length;
        stats.nonEmptyLines = lines.filter(line => line.trim().length > 0).length;
        result = `Total lines: ${stats.totalLines}\nNon-empty lines: ${stats.nonEmptyLines}`;
        break;
        
      case 'uppercase':
        result = content.toUpperCase();
        stats.totalLines = lines.length;
        break;
        
      case 'stats':
        stats.totalLines = lines.length;
        stats.nonEmptyLines = lines.filter(line => line.trim().length > 0).length;
        stats.totalCharacters = content.length;
        stats.words = content.split(/\s+/).filter(word => word.length > 0).length;
        
        result = `File Statistics
===============
Total Lines: ${stats.totalLines}
Non-empty Lines: ${stats.nonEmptyLines}
Total Characters: ${stats.totalCharacters}
Total Words: ${stats.words}
Average Line Length: ${(stats.totalCharacters / stats.totalLines).toFixed(2)}
`;
        break;
        
      default:
        throw new Error(`Unknown operation: ${operation}`);
    }
    
    // Ensure output directory exists
    const outputDir = path.dirname(outputFile);
    await io.mkdirP(outputDir);
    
    // Write output file
    await fs.writeFile(outputFile, result, 'utf8');
    
    core.info(`Output written to: ${outputFile}`);
    
    // Set outputs
    core.setOutput('lines-processed', stats.totalLines || lines.length);
    core.setOutput('output-path', outputFile);
    
    // Log statistics
    if (Object.keys(stats).length > 0) {
      core.startGroup('Processing Statistics');
      Object.entries(stats).forEach(([key, value]) => {
        core.info(`${key}: ${value}`);
      });
      core.endGroup();
    }
    
    // Create summary
    core.summary
      .addHeading('File Processing Complete')
      .addTable([
        [{data: 'Property', header: true}, {data: 'Value', header: true}],
        ['Input File', inputFile],
        ['Output File', outputFile],
        ['Operation', operation],
        ['Lines Processed', (stats.totalLines || lines.length).toString()]
      ])
      .write();
    
  } catch (error) {
    core.setFailed(`Action failed: ${error.message}`);
  }
}

run();
```

7. Click **Commit changes** → **Commit changes**.

8. Create test workflow: Click **Add file** → **Create new file**, name it `.github/workflows/test-file-processor.yml`:

```yaml
name: Test File Processor

on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  process-files:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Create sample file
        run: |
          echo "Line 1: Hello World" > input.txt
          echo "Line 2: GitHub Actions" >> input.txt
          echo "Line 3: Enterprise Training" >> input.txt
      
      - name: Process file - Stats
        id: stats
        uses: ./.github/actions/file-processor
        with:
          input-file: 'input.txt'
          output-file: 'output-stats.txt'
          operation: 'stats'
      
      - name: Display output
        run: cat output-stats.txt
      
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: processed-files
          path: output-*.txt
```

9. Click **Commit changes** → **Commit changes**.

10. Go to **Actions** tab → **Test File Processor** workflow (it runs on push automatically or use Run workflow).

11. View the workflow run and download the processed files artifact.

### What Just Happened?

You created an action that performs file system operations:
- `@actions/io` provides cross-platform file operations like `mkdirP()`, `cp()`, `mv()` that work consistently on Windows, macOS, and Linux
- Native Node.js `fs.promises` API works for reading/writing files when cross-platform behavior isn't critical
- Always check file existence with `fs.access()` before reading to provide clear error messages
- Use `path.dirname()` and `path.join()` for cross-platform path manipulation (handles `/` vs `\`)
- `core.startGroup()` and `core.endGroup()` create collapsible log sections for better readability

Use `@actions/io` for directory creation and file manipulation; use `fs.promises` for reading/writing file content.

---

## Best Practices for JavaScript Actions

### DO
- Use `@actions/core` for inputs, outputs, and logging
- Validate all inputs thoroughly
- Handle errors gracefully with try-catch
- Use semantic logging levels (info, warning, error)
- Create helpful job summaries
- Include comprehensive error messages
- Test actions locally before committing
- Document inputs and outputs clearly

### DON'T
- Log sensitive data (tokens, passwords)
- Assume inputs are always valid
- Use console.log (use core.info instead)
- Ignore error handling
- Create actions that modify the git repository without being explicit
- Make actions too complex (split into multiple actions if needed)

---

## Action Development Checklist

- [ ] `action.yml` is properly configured
- [ ] All required inputs are defined
- [ ] Outputs are clearly defined and set
- [ ] Error handling is comprehensive
- [ ] Logging is helpful and appropriate
- [ ] Dependencies are properly installed
- [ ] Action works on all target platforms
- [ ] Documentation is complete
- [ ] Test workflow validates functionality

---

## Common Issues and Solutions

**Problem:** "Cannot find module '@actions/core'"  
**Solution:** Run `npm install @actions/core` in action directory

**Problem:** "Input required and not supplied"  
**Solution:** Check that input is correctly defined in action.yml and passed in workflow

**Problem:** "Permission denied" errors  
**Solution:** Ensure proper permissions are set in workflow YAML

**Problem:** Action doesn't show outputs  
**Solution:** Verify outputs are set with `core.setOutput()` and defined in action.yml

---

## Key Takeaways

- JavaScript actions run on Node.js runtime
- Use `@actions/core` for core functionality
- Actions should be focused and reusable
- Proper error handling is critical
- Inputs and outputs enable flexibility
- Job summaries improve user experience
- Test thoroughly before deploying

---

## Resources

- [@actions/core documentation](https://github.com/actions/toolkit/tree/main/packages/core)
- [@actions/github documentation](https://github.com/actions/toolkit/tree/main/packages/github)
- [Creating a JavaScript action](https://docs.github.com/en/actions/creating-actions/creating-a-javascript-action)
- [Actions Toolkit](https://github.com/actions/toolkit)

---

**End of Lab 5**
