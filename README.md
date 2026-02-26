# GitHub Actions Workshop (Beginner → Practical)

This workshop teaches how GitHub Actions works by building a tiny console app and progressively automating:
- CI (build + test)
- Releases (build artifacts + GitHub Release)
- Issue automation (labeling based on keywords)

You will create everything step-by-step so you can see what triggers workflows, how jobs run, and how Actions uses permissions.

## Prerequisites

- A GitHub account
- Git installed
- A code editor (VS Code recommended)
- .NET SDK 8 installed locally (for the console app)

You can verify your install with:

```bash
dotnet --version
```

## Workshop outline

1. Create a repository with a simple console application
2. Add a simple Action that builds and tests the console app
3. Improve the Action to create a GitHub Release (and attach build artifacts)
4. Create an Action that labels issues based on keywords in the issue text
5. Enable GitHub Models and summarize issues with an Action
6. (Optional) Add quality gates (lint, formatting), caching, and a workflow badge
7. (Optional) Add environment protection + manual approvals for release

---

## Step 1 — Create the repository + console app

### 1.1 Create a new repository

Create a new GitHub repository (public or private):
- Name: `GHActions-Workshop`
- Add README
- Add .gitignore (Visual Studio)
- Add license (optional)

Clone it:

```bash
git clone https://github.com/<YOUR_ORG_OR_USERNAME>/GHActions-Workshop.git
cd GHActions-Workshop
```

### 1.2 Create a tiny .NET console app (+ unit tests)

Create a solution with a console app and an xUnit test project:

```bash
dotnet new sln -n GHActionsWorkshop
dotnet new console -n WorkshopApp
dotnet new xunit -n WorkshopApp.Tests

dotnet sln GHActionsWorkshop.sln add WorkshopApp/WorkshopApp.csproj
dotnet sln GHActionsWorkshop.sln add WorkshopApp.Tests/WorkshopApp.Tests.csproj

dotnet add WorkshopApp.Tests/WorkshopApp.Tests.csproj reference WorkshopApp/WorkshopApp.csproj
```

Add a tiny function to test in [WorkshopApp/Calculator.cs](WorkshopApp/Calculator.cs):

```csharp
namespace WorkshopApp;

public static class Calculator
{
    public static int Sum(int a, int b) => a + b;
}
```

Update the console entrypoint in [WorkshopApp/Program.cs](WorkshopApp/Program.cs):

```csharp
using WorkshopApp;

var a = args.Length > 0 ? int.Parse(args[0]) : 1;
var b = args.Length > 1 ? int.Parse(args[1]) : 2;

Console.WriteLine($"{a} + {b} = {Calculator.Sum(a, b)}");
```

Run the console to check if the code works (run the command in the same folder of the console project):

```bash
dotnet run --project WorkshopApp -- 3 4
```

Add a minimal test in [WorkshopApp.Tests/CalculatorTests.cs](WorkshopApp.Tests/CalculatorTests.cs):

```csharp
using WorkshopApp;
using Xunit;

namespace WorkshopApp.Tests;

public class CalculatorTests
{
    [Fact]
    public void Sum_AddsNumbers()
    {
        Assert.Equal(5, Calculator.Sum(2, 3));
    }
}
```

Run locally:

```bash
dotnet build
dotnet test
dotnet run --project WorkshopApp -- 3 4
```

Commit and push:

```bash
git add -A
git commit -m "Add simple console app + tests"
git push
```

What you learned:
- You now have something deterministic to build and test in CI.

---

## Step 2 — Add a basic CI workflow (build + test)

Create a workflow file at [.github/workflows/ci.yml](.github/workflows/ci.yml).

### 2.1 What a workflow is

A workflow is a YAML file that defines:
- **Triggers** (`on:`) such as push, pull_request, schedule, workflow_dispatch
- **Jobs** (each job runs on a runner VM)
- **Steps** inside each job (run shell commands or use reusable Actions)

### 2.2 Add the CI workflow

Create [.github/workflows/ci.yml](.github/workflows/ci.yml):

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --verbosity normal
```

Commit and push:

```bash
git add -A
git commit -m "Add CI workflow (build + test)"
git push
```

Verify:
- Go to the repository → **Actions** tab
- You should see a workflow run for your push
- Open the run and inspect each step’s logs

What you learned:
- Triggers cause runs
- Jobs are isolated environments (fresh machine each run)
- Steps can use Actions from the Marketplace

---

## Step 3 — Add a release workflow (create a GitHub Release + upload artifacts)

Goal: when you tag a version like `v1.0.0`, GitHub Actions will:
- run tests
- build a distributable artifact
- create a GitHub Release
- attach the artifact to the release

### 3.1 Decide what “release artifact” means

For .NET, a common pattern is:
- `dotnet publish` to produce a self-contained folder of compiled output
- Zip that folder
- Attach the zip to a GitHub Release

### 3.2 Create the release workflow

Create [.github/workflows/release.yml](.github/workflows/release.yml):

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --verbosity normal

      - name: Publish
        run: dotnet publish WorkshopApp/WorkshopApp.csproj --configuration Release --no-build --output dist

      - name: Create zip
        run: |
          cd dist
          zip -r "../WorkshopApp-${{ github.ref_name }}.zip" .

      - name: Upload artifact to workflow run
        uses: actions/upload-artifact@v4
        with:
          name: WorkshopApp-${{ github.ref_name }}
          path: WorkshopApp-${{ github.ref_name }}.zip

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: |
            WorkshopApp-${{ github.ref_name }}.zip
          generate_release_notes: true
```

Why `permissions` matters:
- Creating a release modifies repository contents metadata.
- The default token (`GITHUB_TOKEN`) needs `contents: write` for release creation.

### 3.3 Create a release by tagging

Commit the workflow:

```bash
git add -A
git commit -m "Add release workflow (tag → GitHub Release)"
git push
```

Create a tag and push it:

```bash
git tag v0.1.0
git push origin v0.1.0
```

Verify:
- Actions tab shows a run for the tag push
- Releases page contains a new release with auto-generated notes
- The `WorkshopApp-<tag>.zip` file is attached

What you learned:
- Tag triggers are commonly used for releases
- Permissions must be explicit for secure automation
- Uploading artifacts helps debugging and distribution

---

## Step 4 — Auto-label issues based on keywords

Goal: when someone opens an issue (or edits it), a workflow will apply labels such as:
- `bug` if the issue contains “bug”, “error”, “crash”, “exception”
- `feature` if it contains “feature”, “enhancement”, “request”
- `docs` if it contains “docs”, “documentation”, “readme”

### 4.1 Create labels in the repository

In GitHub UI:
- Repository → Issues → Labels
- Create labels: `bug`, `feature`, `docs`

(Names must match exactly.)

### 4.2 Add the labeling workflow

Create [.github/workflows/issue-labeler.yml](.github/workflows/issue-labeler.yml):

```yaml
name: Issue auto-label

on:
  issues:
    types: [opened, edited]

permissions:
  issues: write

jobs:
  label:
    runs-on: ubuntu-latest

    steps:
      - name: Apply labels based on keywords
        uses: actions/github-script@v7
        with:
          script: |
            const issue = context.payload.issue;
            const text = `${issue.title}\n\n${issue.body || ''}`.toLowerCase();

            const labelRules = [
              { label: 'bug', keywords: ['bug', 'error', 'crash', 'exception', 'stack trace', 'fails'] },
              { label: 'feature', keywords: ['feature', 'enhancement', 'request', 'proposal'] },
              { label: 'docs', keywords: ['docs', 'documentation', 'readme', 'typo'] },
            ];

            const labelsToAdd = [];
            for (const rule of labelRules) {
              if (rule.keywords.some(k => text.includes(k))) {
                labelsToAdd.push(rule.label);
              }
            }

            if (labelsToAdd.length === 0) {
              core.info('No labels matched');
              return;
            }

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue.number,
              labels: labelsToAdd,
            });

            core.info(`Added labels: ${labelsToAdd.join(', ')}`);
```

Commit and push:

```bash
git add -A
git commit -m "Add issue auto-label workflow"
git push
```

Verify:
- Open a new issue like: “Bug: app crash when sum runs”
- The issue should automatically get labeled `bug`

What you learned:
- Workflows can be triggered by issue events
- The GitHub API is accessible from Actions via `actions/github-script`
- Fine-grained permissions are safer than broad permissions

---

## Step 5 — Enable GitHub Models + summarize issues with an Action

Goal: when someone opens an issue (or edits it), GitHub Actions will call a GitHub Model to generate a short summary and post it as a comment.

### 5.1 Enable GitHub Models for your repo/account

GitHub Models is an AI inference API you can call using GitHub credentials.

Do this to confirm it’s available:

1. Go to https://github.com/marketplace/models and try a model in the playground.
2. In your repository, check if you have a **Models** tab.

If you don’t see the Models tab:
- Your organization/enterprise admin may need to enable GitHub Models access for your org, or it may not be available for your plan.

Important for Actions:
- Workflows that call GitHub Models must grant `models: read` in `permissions:`.

### 5.2 Add a workflow that summarizes issues

Create [.github/workflows/issue-summarizer.yml](.github/workflows/issue-summarizer.yml):

```yaml
name: Issue summarizer (GitHub Models)

on:
  issues:
    types: [opened, edited]

permissions:
    issues: write
    models: read
    contents: read

jobs:
  summarize:
    runs-on: ubuntu-latest

    steps:
      - name: Run AI inference
        id: inference
        uses: actions/ai-inference@v1
        with:
          prompt: |
            Summarize the following GitHub issue in one paragraph:
            Title: ${{ github.event.issue.title }}
            Body: ${{ github.event.issue.body }}

      - name: Comment with AI summary
        run: |
          gh issue comment "$ISSUE_NUMBER" --body "$RESPONSE"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          RESPONSE: ${{ steps.inference.outputs.response }}
```

Commit and push:

```bash
git add -A
git commit -m "Add issue summarizer workflow (GitHub Models)"
git push
```

Verify:
- Open or edit an issue
- The workflow should run
- A new comment should be added with the summary

What you learned:
- New permission type: `models: read`
- Workflows can call AI models securely using `GITHUB_TOKEN`
- You can combine model output with GitHub API automation (commenting)

---

## Step 6 (Optional) — Add polish that helps learners

### 6.1 Add a workflow badge to the README

In GitHub, open the CI workflow, click “…” → “Create status badge”, then paste it into this README.

### 6.2 Add linting

Add formatting checks (and run them in CI):
- `dotnet format` (formatting)
- Treat warnings as errors (quality gate)

Example step to add to CI:

```yaml
- name: Format (verify)
  run: dotnet format --verify-no-changes
```

### 6.3 Use a matrix build

Update CI to test multiple OSes (common for .NET tooling differences):

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
```

Then use `runs-on: ${{ matrix.os }}`.

---

## Step 7 (Optional) — Protect releases

Teach the concept of environments and approvals:
- Create an environment named `production`
- Require approval
- Modify the release workflow to deploy only after approval

This demonstrates safe release practices for real teams.

---

## Troubleshooting checklist

- Workflow didn’t run: check the `on:` trigger and branch/tag names.
- Permission error (403): check the workflow `permissions:` block.
- Release didn’t publish: verify the tag format matches `v*`.
- Issue labeling didn’t happen: ensure the workflow has `issues: write` and labels exist.
- .NET build fails in Actions but works locally: confirm the workflow uses the right `dotnet-version`.

---

## Discussion prompts (for instructors)

- What is the difference between a workflow, job, and step?
- Why use `pull_request` triggers in addition to `push`?
- What does `GITHUB_TOKEN` do and why do permissions matter?
- Why are tag-triggered releases safer than “release on every push to main”?
- Which parts of your pipeline should be required checks for merging PRs?
