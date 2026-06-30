# Create New Project

Create a fresh copy of this scaffold as a new project, with its own private GitHub repo under either the user's GitHub account or an accessible organization.

## Instructions for AI

**IMPORTANT: Always ask for the project name first before doing anything else!**

> **internalsphere auto-detection:** This command supports two flows. If the destination GitHub org is
> **`internalsphere`**, the repo is managed by the `internal-app-orchestrator` and you MUST use the
> **internalsphere flow** (step 9b) instead of pushing the scaffold directly. Pushing a full scaffold
> onto a fresh internalsphere repo collides with the orchestrator baseline (divergent history, a
> policy-rejected `ci.yml`, a missing lockfile, and a 404 from a missing framework preset). For any
> other destination, use the **standard flow** (step 9a). See [`docs/internalsphere.md`](../../docs/internalsphere.md)
> for the why.

## Prerequisites

- `gh` CLI must be installed and authenticated (`gh auth status`)
- If not authenticated, instruct the user to run `gh auth login` first
- The authenticated GitHub account must have permission to create repositories in the selected destination account or organization
- If expected organizations or teams do not appear, the user may need to refresh `gh` permissions with `gh auth refresh -s read:org`

## Steps

1. **Ask for project name FIRST** - Display this prompt:

   ***

   ## 🆕 Let's create your new project!

   **What would you like to name your project?**

   Some tips for a good name:
   - Use lowercase letters
   - Use dashes instead of spaces (e.g., `my-cool-app`)
   - Keep it short and memorable
   - Examples: `todo-app`, `my-portfolio`, `awesome-project`

   **Type your project name below:**

   ***

   **WAIT for the user to respond with a name before proceeding!**

2. **Validate the project name** when user provides it:
   - If name has spaces, convert to dashes and confirm: "I'll use `{converted-name}` - is that okay?"
   - If name has uppercase, convert to lowercase and confirm
   - If name has special characters (except dashes), ask them to choose a different name

3. **Ask where to create it locally** (optional):
   - Default: parent directory of current project
   - Ask: "I'll create this in `{parent-directory}`. Is that okay, or would you like a different location?"

4. **Ask where to create the GitHub repo**:
   - First verify GitHub authentication:
     ```bash
     gh auth status
     ```
   - Get the authenticated user's login:
     ```bash
     gh api user --jq .login
     ```
   - Get organizations available to the authenticated user:
     ```bash
     gh api user/orgs --paginate --jq '.[].login'
     ```
   - Present the options clearly:
     - Personal account: `<github-username>/<project-name>`
     - Each available organization: `<org-login>/<project-name>`
   - Ask: "Where should I create the private GitHub repo?"
   - If no organizations are returned, explain that the project can be created under the personal account unless they authenticate with an account that has org access or refresh `gh` permissions with `gh auth refresh -s read:org`.
   - Explain that the org list shows accessible organizations, but the final repo creation still depends on that org allowing the user to create repositories.
   - If the user selects an organization, optionally ask whether a GitHub team should get access:
     - List available teams with:
       ```bash
       gh api orgs/<org-login>/teams --paginate --jq '.[].slug'
       ```
     - If the team list command fails, continue without a team and mention that the org may require additional `gh` permissions such as `read:org`.
     - If the user selects a team, remember the team slug for the repo creation command.
   - **Detect internalsphere:** if the chosen org is **`internalsphere`**, set the flow to the
     **internalsphere flow (step 9b)** and tell the user this is a managed app that deploys via the
     orchestrator. Otherwise use the **standard flow (step 9a)**.

5. **Copy the scaffold** - Set a source path for the cleaned scaffold copy:

   **Standard flow** (personal accounts and non-internalsphere orgs):
   ```bash
   scaffold_source="<destination>/<project-name>"
   cp -r "$(pwd)" "$scaffold_source"
   ```

   **internalsphere flow** (when chosen GitHub org is `internalsphere`):
   ```bash
   scaffold_source="<destination>/<project-name>-src"
   cp -r "$(pwd)" "$scaffold_source"
   ```

   For internalsphere, the final `<destination>/<project-name>` folder will be a clone of the
   orchestrator-managed repo (created in step 9b), so the cleaned scaffold source must live at
   `<project-name>-src`.

6. **Clean up the new project**:

   ```bash
   cd "$scaffold_source"
   rm -f .cursor/commands/new-project.md
   rm -f .cursor/commands/update.md
   rm -f .cursor/commands/bump-scaffold.md
   rm -f .cursor/rules/scaffold-protection.mdc
   rm -rf node_modules .git pnpm-lock.yaml package-lock.json
   rm -f scripts/bump-scaffold.sh
   rmdir scripts 2>/dev/null || true
   ```

   > **Keep** `vercel.json`, `docs/internalsphere.md`, and `.cursor/rules/internalsphere.mdc` — these
   > are real project files (not scaffold-maintenance files) and must carry over to the new project.

7. **Update copied scaffold metadata**:
   - Make these edits inside `$scaffold_source`
   - Change the `"name"` field in `package.json` to the new project name
   - Remove the `"bump:scaffold"` script from `package.json`
   - Update `.cursor/commands/help.md` to remove `/new-project`, `/update`, and `/bump-scaffold` sections plus any scaffold-maintainer workflow text
   - If the copied project keeps the starter `README.md`, remove scaffold-only command references there as well
   - Make sure the new project does not advertise scaffold-only maintenance commands or include scaffold-protection prompts

8. **Initialize fresh git repo** (standard flow only — the internalsphere flow gets its git history by
   cloning the managed repo in step 9b, so skip this step for internalsphere):

   ```bash
   git init
   git add .
   git commit -m "Initial commit from scaffold"
   ```

9a. **Standard flow — create a private GitHub repo and push** (use this for personal accounts and any
org that is **not** `internalsphere`):

- For a personal repo:
  ```bash
  gh repo create <github-username>/<project-name> --private --source . --push
  ```
- For an organization repo:
  ```bash
  gh repo create <org-login>/<project-name> --private --source . --push
  ```
- For an organization repo with team access:
  ```bash
  gh repo create <org-login>/<project-name> --private --team <team-slug> --source . --push
  ```
- This creates a private repo, sets it as `origin`, and pushes the initial commit
- If the repo name is already taken, append a suffix or ask the user for an alternative
- Do **not** run `vercel deploy` or create a Vercel project as part of `/new-project`

9b. **internalsphere flow** (use this when the chosen org is **`internalsphere`**). The
`internal-app-orchestrator` owns Vercel, CI, branch protection, and the baseline files — so you
create an **empty** repo, let it bootstrap, and then add the scaffold's app code via a PR. Full
detail and rationale: [`docs/internalsphere.md`](../../docs/internalsphere.md).

1.  **Create an empty private repo** (no `--source`, no `--push`):
    ```bash
    gh repo create internalsphere/<project-name> --private
    ```
    If the user selected a GitHub team in step 4, grant that team access when creating the repo:
    ```bash
    gh repo create internalsphere/<project-name> --private --team <team-slug>
    ```
2.  **Wait for the orchestrator to bootstrap it** (~1–2 min). It seeds `app-manifest.yml`,
    `.sops.yaml`, `secrets/`, `.github/workflows/managed-app.yml`, Cursor skills, `QUICKSTART.md`,
    and a baseline `vercel.json`, then auto-merges its bootstrap PR to `main`. Poll until `main` has
    the baseline marker file:

    ```bash
    for attempt in {1..24}; do
      if gh api repos/internalsphere/<project-name>/contents/app-manifest.yml >/dev/null 2>&1; then
        echo "internalsphere bootstrap is ready"
        break
      fi

      if [ "$attempt" -eq 24 ]; then
        echo "Timed out waiting for internalsphere bootstrap; ask in #proj-internalsphere"
        exit 1
      fi

      echo "Waiting for internalsphere bootstrap..."
      sleep 10
    done
    ```

3.  **Clone the bootstrapped repo** into the destination and run one-time setup:
    ```bash
    git clone https://github.com/internalsphere/<project-name>.git "<destination>/<project-name>"
    cd "<destination>/<project-name>"
    sh scripts/setup-repo.sh
    ```
4.  **Add the scaffold app code on a branch.** Copy the cleaned app code from the `-src` copy you made
    in step 5 into the clone — `src/`, `public/`, `prisma/`, and root config like `package.json`,
    `tsconfig*.json`, `next.config.ts`, `eslint.config.mjs`, `postcss.config.mjs`,
    `tailwind.config.ts`, `components.json`, `.cursor/rules/` (incl. `internalsphere.mdc`),
    `.cursor/commands/`, and `docs/`. **Do NOT copy or overwrite orchestrator-managed files**: any
    `.github/workflows/*`, `CODEOWNERS`, `.sops.yaml`, `secrets/`, git hooks, `scripts/`,
    `QUICKSTART.md`. **Do NOT add a `.github/workflows/ci.yml`** (it fails the `ci-required` policy
    check). Delete the `-src` copy when done.
5.  **Merge `framework: nextjs` into the orchestrator's `vercel.json`** (don't overwrite it — keep
    its `git.deploymentEnabled: false`). The result should be:
    ```json
    { "framework": "nextjs", "git": { "deploymentEnabled": false } }
    ```
6.  **Install to generate the lockfile, then commit and open a PR:**
    ```bash
    pnpm install
    git checkout -b add-app-code
    git add -A
    git commit -m "Add app scaffold"
    git push -u origin add-app-code
    gh pr create --base main --fill
    ```
7.  **Merge when green.** The `internalsphere-ranger` bot posts the preview URL on the PR; merging to
    `main` triggers the production deploy. Do **not** run `vercel deploy` yourself.

8.  **Display success message**:

For a personal repo, use:

```bash
https://github.com/<github-username>/<project-name>
```

For an organization repo, use:

```bash
https://github.com/<org-login>/<project-name>
```

---

## ✅ Your new project "{project-name}" is ready!

**Project location:** `<full-path-to-new-project>`
**GitHub repo:** `<github-repo-url>`

### 👉 Next Steps

1.  **Open your new project in Cursor:**
    - Press `Cmd+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows)
    - Type "Open Folder" and select it
    - Navigate to: `<full-path-to-new-project>`
    - Click "Open"
    - Important: keep working from this new folder, not the scaffold folder

2.  **Once you're in the new project, run `/setup`** to install dependencies

3.  **Then run `/start`** to launch your app!

**Standard flow:** No Vercel deployment was created. This only created a local project and private
GitHub repo.

**internalsphere flow:** The orchestrator owns deploys — your app goes live automatically when the
app-code PR merges to `main`. Find the project and its canonical URL under
`https://vercel.com/anysphere-internal/<project-name>/deployments`. See
[`docs/internalsphere.md`](../../docs/internalsphere.md) for the full workflow and the 404 checklist.

---

## Important Reminders

- **DO NOT proceed without getting a project name from the user first**
- Wait for user input at step 1 before running any commands
- Make sure to use the exact name the user provides (after validation/conversion)
- The GitHub repo is always created as **private** by default
- Always ask whether the repo should be created under the personal GitHub account or an accessible organization
- **If the chosen org is `internalsphere`, use the internalsphere flow (step 9b) — never push the scaffold onto a fresh internalsphere repo**
- Make it very clear that the user must open the new project folder in Cursor after creation
- Do not run `vercel deploy` yourself in either flow (internalsphere deploys happen via the orchestrator on merge)

## Tone

Be helpful and encouraging! Creating a new project is exciting. Guide them through each step clearly and celebrate when it's done.
