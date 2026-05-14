Publish updated documentation to Confluence.

## Step 1 — Find recently changed docs

Run this command to get all markdown files changed in the last git commit, excluding files under `.claude/`:

```
git diff --name-only HEAD~5 HEAD -- '*.md' | grep -v '^\.claude/'
```

Also check for any locally modified (unstaged/staged) markdown files:

```
git diff --name-only -- '*.md' | grep -v '^\.claude/'
git diff --cached --name-only -- '*.md' | grep -v '^\.claude/'
```

Combine and deduplicate the results into a final list of candidate files.

## Step 2 — Ask for exclusions

Present the list to the user clearly, then ask:

> "The following documents have recent changes and will be published to Confluence:
> [list each file]
>
> Are there any you'd like to exclude? If not, just say 'proceed'."

Wait for the user's response before continuing.

## Step 3 — Publish

After the user confirms (with or without exclusions), run `md2confluence <file>` for each remaining file **sequentially**. Each command must succeed before proceeding to the next. If any command fails, stop immediately and report the error with the file that failed.

## Step 4 — Confirm

After all commands complete successfully, report how many documents were published and list them.
