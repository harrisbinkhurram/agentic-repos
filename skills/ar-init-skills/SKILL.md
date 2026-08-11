---
name: ar-init-skills
description: Initialize skills configuration for this project. Use when setting up for first time or when skill.config is missing. Say "ar-init-skills" or "setup skills".
disable-model-invocation: true
---

# Initialize Skills

Set up user-specific configuration for Claude skills.

## When to Use

- First time using skills in this project
- `skill.config` is missing
- New developer joining project

## Standard Conventions (Enforced Internally)

**IMPORTANT:** These conventions are standardized across all users. Do NOT ask users for these - derive them automatically.

### Directory Structure

```
{coding_tasks_root}/                   # User provides this (e.g., .../{project_name}_Coding_Tasks)
├── Backend/                           # Backend platform
│   ├── OnGoingTasks/                  # Auto-created
│   └── DoneTasks/                     # Auto-created
├── Frontend/                          # Frontend Web platform
│   ├── OnGoingTasks/
│   └── DoneTasks/
├── iOS_Frontend/                      # iOS Frontend platform
│   ├── OnGoingTasks/
│   └── DoneTasks/
├── Android_Frontend/                  # Android Frontend platform
│   ├── OnGoingTasks/
│   └── DoneTasks/
├── TasksSummary/                      # Auto-created
│   ├── Backend.md                     # Auto-created based on platform
│   ├── Frontend.md                    # For web frontend
│   ├── iOS_Frontend.md                # For iOS frontend
│   └── Android_Frontend.md            # For Android frontend
└── WeeklySummaries/                   # Auto-created
    └── (weekly summary files organized by date)

{docs_root}/                           # User provides this (optional)
├── PublishedDocs/Engineering/Templates/    # Standard templates location
└── AI_Workflows/SkillUpdates/              # Standard skill updates location
```

### Platform Types

Supported platforms and their directory names:
- **Backend** → Directory: `Backend/`
- **Frontend Web** → Directory: `Frontend/`
- **iOS Frontend** → Directory: `iOS_Frontend/`
- **Android Frontend** → Directory: `Android_Frontend/`

### Path Derivation Rules

**Only store in config:**
- `vault_root` (user-provided — the coding-tasks root itself, the parent of the platform folders)
- `tasks_root` (user-provided, e.g., `/path/to/Backend`)
- `docs_root` (user-provided, optional)
- `platform` (auto-detected and stored for quick reference)

**Read from config, never derive:**
- `coding_tasks_root` = `{vault_root}`

  Do **not** compute it as `dirname({tasks_root})`. That is correct only while `tasks_root`
  sits exactly one level below the vault. A workspace that groups tasks by product before
  platform (`{vault}/{Product}/{Platform}`) puts it two levels down, so `dirname` returns the
  *product* folder — and every product's `TasksSummary`, `WeeklySummaries`, and review
  artifacts then get written under whichever single product happens to be configured. The user
  already supplies the vault root in step 3; store it rather than recovering it later.

**Auto-derive at runtime:**
From `tasks_root`:
- `tasks_folder` = `{tasks_root}/OnGoingTasks`
- `done_folder` = `{tasks_root}/DoneTasks`
- `platform` = `basename({tasks_root})` (also stored in config)

From `vault_root`:
- `task_summary_folder` = `{vault_root}/TasksSummary`
- `weekly_summaries_folder` = `{vault_root}/WeeklySummaries`

From `docs_root` (if provided):
- `templates_folder` = `{docs_root}/PublishedDocs/Engineering/Templates`
- `skill_updates_folder` = `{docs_root}/AI_Workflows/SkillUpdates`

## Steps

### 1. Check Prerequisites

```bash
cat .claude/skill.config 2>/dev/null || echo "CONFIG_MISSING"
```

**If skill.config exists, check version against framework:**
```bash
current=$(jq -r '._schema_version // ""' .claude/skill.config 2>/dev/null)
expected=$(jq -r '.framework_version // ""' .claude/config_hints.json 2>/dev/null)
```

If `current == expected`: config is current, tell user and stop (unless they explicitly want to reconfigure).
If `current != expected`: config is outdated, continue with setup to update it. At the end, set `_schema_version` to match `framework_version`.

**Check GitHub CLI:**
```bash
command -v gh >/dev/null 2>&1 && echo "GH_OK" || echo "GH_MISSING"
```

If `gh` is missing:
```
GitHub CLI (gh) is recommended for creating PRs and GitHub integration.

Install:
  macOS:    brew install gh
  Ubuntu:   sudo apt install gh
  Windows:  winget install GitHub.cli

Then authenticate:  gh auth login

Install now or skip? (install/skip)
```

If skip, continue, skills that need `gh` will warn at runtime.
If install, wait for user and re-check.

If `gh` exists but not authenticated (`gh auth status` fails):
```
gh is installed but not logged in. Run:  gh auth login
```

### 2. Ask for Platform Type

Ask user to select their platform:

```
Setting up skill configuration.

What platform are you working on?

1. Backend
2. Frontend Web
3. iOS Frontend
4. Android Frontend

Select (1-4):
```

Map user selection to directory name:
- `1` → Platform: `Backend` → Directory: `Backend/`
- `2` → Platform: `Frontend` → Directory: `Frontend/`
- `3` → Platform: `iOS_Frontend` → Directory: `iOS_Frontend/`
- `4` → Platform: `Android_Frontend` → Directory: `Android_Frontend/`

### 3. Ask for Tasks Root Path

Ask user for the coding tasks root path:

```
I need the path to your coding tasks directory.

This should be the parent folder that contains all platform folders.
Example: /path/to/{project_name}_Coding_Tasks

The platform folder "{selected_platform}" will be created inside this directory.

Tip: Make this folder a Git repo to track task history and collaborate with your team.

What is your coding tasks root path?
```

After user provides the path, construct `tasks_root`:
```bash
tasks_root="{user_provided_path}/{platform_directory_name}"
# Example: /path/to/your-project-coding-tasks/Backend
# Example: /path/to/your-project-coding-tasks/Frontend
# Example: /path/to/your-project-coding-tasks/iOS_Frontend
```

### 4. Create Platform Structure

After user provides the path and platform is selected:

1. **Use platform from user selection** (already determined in step 2):
   ```bash
   # Platform variable already set from user selection
   # Examples: "Backend", "Frontend", "iOS_Frontend", "Android_Frontend"
   ```

2. **Construct tasks_root**:
   ```bash
   tasks_root="{user_provided_coding_tasks_root}/{platform}"
   coding_tasks_root="{user_provided_coding_tasks_root}"
   ```

3. **Create platform task directories**:
   ```bash
   mkdir -p "$tasks_root/OnGoingTasks"
   mkdir -p "$tasks_root/DoneTasks"
   ```

4. **Create TasksSummary, WeeklySummaries, and CodeReviews root folders**:
   ```bash
   mkdir -p "$coding_tasks_root/TasksSummary"
   mkdir -p "$coding_tasks_root/WeeklySummaries"
   mkdir -p "$coding_tasks_root/CodeReviews"
   ```

   **Gitignore CodeReviews** (local review artifacts, never committed):
   ```bash
   if ! grep -q "CodeReviews/" "$coding_tasks_root/.gitignore" 2>/dev/null; then
     echo "CodeReviews/" >> "$coding_tasks_root/.gitignore"
   fi
   ```

   **Note:** WeeklySummaries will contain week-specific folders (created by ar-taskflow):
   - Format: `WeeklySummaries/YYYY-MM-DD/` (Friday dates)
   - Example: `WeeklySummaries/2026-01-30/{namespace}-195.md`

5. **Create platform-specific summary file** if it doesn't exist:
   ```bash
   summary_file="$coding_tasks_root/TasksSummary/{platform}.md"
   if [ ! -f "$summary_file" ]; then
     echo "# {platform} Tasks Summary" > "$summary_file"
   fi
   ```

### 5. Ask for Docs Path (Optional)

```
Do you have a separate documentation project?
If yes, provide the path. If no, just say "skip".

Example: /path/to/{project_name}_DocsProject

This is optional but recommended for maintaining engineering specs and templates.
```

If user provides path:
- Derive `templates_folder` and `skill_updates_folder` automatically

If user says skip/no:
- Set docs paths to empty/null

### 6. Create skill.config

Based on user input, create `.claude/skill.config`:

```json
{
  "_user": "{git_user_email}",
  "_schema_version": "{framework_version}",
  "paths": {
    "vault_root": "{user_provided_coding_tasks_root}",
    "tasks_root": "{constructed_tasks_root}",
    "docs_root": "{user_provided_docs_path_or_null}"
  }
}
```

`vault_root` is the path the user gave in step 3. Store it — do not make later skills recover
it with `dirname(tasks_root)` (see "Path Derivation Rules" for why that derivation is wrong
once a workspace nests deeper than `{vault}/{Platform}`).

**This step MUST NOT overwrite an existing config.** If `.claude/skill.config` already exists,
merge into it and preserve every key you did not ask about. A later framework version, or the
user, may have added keys this step knows nothing about; rewriting the file wholesale discards
them silently, and the user only finds out when behaviour they had already fixed comes back.

```bash
if [ -f .claude/skill.config ]; then
  # Merge: values collected in this run win; every other key is preserved as-is.
  jq --arg vault "$coding_tasks_root" \
     --arg tasks "$tasks_root" \
     --arg docs "$docs_root" \
     --arg ver "$expected" \
     '._schema_version = $ver
      | .paths.vault_root = $vault
      | .paths.tasks_root = $tasks
      | .paths.docs_root = (if $docs == "" then .paths.docs_root else $docs end)' \
     .claude/skill.config > .claude/skill.config.tmp \
    && mv .claude/skill.config.tmp .claude/skill.config
else
  # Fresh install: write the template above.
  :
fi
```

`$expected` is the framework version read in step 1 — pass that, not an unset variable, or
`_schema_version` lands empty and step 1's staleness check then re-runs setup on every future
invocation. Afterwards confirm the result parses (`jq -e . .claude/skill.config`) and report
which keys were preserved rather than rewritten.

**Note:** No comments or preferences in user's config - those are internal concerns in config_hints.json.

### 7. Update config_hints.json

Update `.claude/config_hints.json` with detected platform:

```json
{
  "_comment": "Project-specific configuration hints. Can be committed to git.",
  "_schema_version": "{framework_version}",

  "platform": "{detected_platform}",
  "standards_location": "{detected_standards_dir}",

  "project": {
    "namespace": "{detected_namespace}",
    "name": "{detected_project_name}",
    "tracker": { "type": "{detected_tracker_type}", "url": "{detected_tracker_url}" }
  },

  "_platform_options": ["java-spring-boot", "react", "ruby-rails", "go", "python"],

  "path_derivation_rules": {
    "_comment": "How to derive paths from skill.config at runtime (for documentation only)",
    "coding_tasks_root": "skill.config paths.vault_root — read it, never dirname(tasks_root)",
    "tasks_folder": "{tasks_root}/OnGoingTasks",
    "done_folder": "{tasks_root}/DoneTasks",
    "task_summary_folder": "{coding_tasks_root}/TasksSummary",
    "weekly_summaries": "{coding_tasks_root}/WeeklySummaries/{YYYY-MM-DD}",
    "weekly_summaries_root": "{coding_tasks_root}/WeeklySummaries",
    "templates_folder": "{docs_root}/PublishedDocs/Engineering/Templates",
    "skill_updates_folder": "{docs_root}/AI_Workflows/SkillUpdates"
  }
}
```

**Purpose:**
- Stores project-level metadata (platform) that's the same for all users
- Contains NO absolute paths (can be committed to git)
- Documents path derivation rules for skills
- Skills compute actual paths at runtime from skill.config

### 8. Update .gitignore

```bash
# Only ignore skill.config (user-specific)
grep -q "skill.config" .gitignore || echo ".claude/skill.config" >> .gitignore

# config_hints.json IS committed (project-level metadata)
```

### 9. Setup MCP Servers (Optional)

```
To enable Jira and Confluence integration, you can set up MCP servers.
This allows starting tasks from Jira tickets and fetching ticket details.

Would you like to set this up now? (y/n)
```

If yes → Run: `claude mcp add --scope user --transport http atlassian https://mcp.atlassian.com/v1/mcp`
If no → Mention they can run "ar-init-mcps" anytime

### 10. Confirm

```
Skills initialized!

Created: .claude/skill.config

Configuration:
  Platform: {platform}
  Coding Tasks Root: {coding_tasks_root}
    - Tasks: {tasks_root}
      - OnGoing: OnGoingTasks/
      - Done: DoneTasks/
    - Summary: TasksSummary/{platform}.md
    - Weekly: WeeklySummaries/
  Docs: {docs_root or "Not configured"}

Next steps:
  1. Run "ar-init-mcps" to set up Jira/Confluence integration (optional)
  2. Run "ar-taskflow" to begin a new task

You're all set!
```

## Reading Config (For Other Skills)

**Two-file approach:**

1. **skill.config** - User-specific paths (NOT committed)
2. **config_hints.json** - Project metadata (IS committed, no absolute paths)

**Reading configuration:**

```bash
# Read user config (absolute paths)
if [ ! -f ".claude/skill.config" ]; then
  echo "Run 'ar-init-skills' first"
  exit 1
fi

tasks_root=$(jq -r '.paths.tasks_root' .claude/skill.config)
docs_root=$(jq -r '.paths.docs_root' .claude/skill.config)

# Read project hints (platform, derivation rules)
platform=$(jq -r '.platform' .claude/config_hints.json)

# Read the vault root. Do NOT use dirname "$tasks_root" — see the warning below.
coding_tasks_root=$(jq -r '.paths.vault_root // ""' .claude/skill.config)
[ -z "$coding_tasks_root" ] && coding_tasks_root=$(dirname "$tasks_root")

tasks_folder="$tasks_root/OnGoingTasks"
done_folder="$tasks_root/DoneTasks"
task_summary_folder="$coding_tasks_root/TasksSummary"
weekly_summaries_folder="$coding_tasks_root/WeeklySummaries"
```

**Do not copy `coding_tasks_root=$(dirname "$tasks_root")` into a skill.** It holds only when
`tasks_root` is exactly one level below the vault. A workspace nested as
`{vault}/{Product}/{Platform}` makes that line return the *product* folder, so every product's
`TasksSummary`, `WeeklySummaries`, and review artifacts collapse under one product. Read
`paths.vault_root` instead. The `dirname` above is a fallback for configs written before that
key existed, so existing installs keep working until they next re-run this skill.

This block is the snippet other skills copy from, so a wrong line here propagates. Keep it
correct.

**Benefits:**
- skill.config = user-specific, never committed
- config_hints.json = project-level, committed to git
- No absolute paths in git
- Each user has their own paths via skill.config
- All users share same platform metadata via config_hints.json

## Adding New Platforms (Future)

When a new platform is needed (e.g., Android_Frontend):
1. User creates tasks folder: `.../project-coding-tasks/Android_Frontend`
2. Run "ar-init-skills" with that path
3. Platform is auto-detected from folder name
4. Summary file `Android_Frontend.md` is auto-created

Supported platforms:
- Backend
- Frontend
- Android_Frontend (future)
- iOS_Frontend (future)
