# Tutorial: How to Archive Your Pull Request Comments

This guide explains how to use a set of scripts to find all the pull requests you have commented on in a repository and archive those comments into a single, clean Markdown file.

This is useful for performance reviews, work logs, or simply keeping a personal record of your contributions.

## Overview

The process uses three scripts that work together:

1.  `get_commented_prs.sh`: Finds all the PRs a specific user has commented on.
2.  `get_pr_comments.sh`: Fetches all the comments from a single PR for that user.
3.  `archive_user_comments.sh`: The main script that uses the other two to generate the final archive file.

## Prerequisites

Before you begin, you must have the following command-line tools installed:

1.  **GitHub CLI (`gh`)**: For interacting with the GitHub API.
    - [Installation Guide](https://github.com/cli/cli#installation)
2.  **jq**: A lightweight and flexible command-line JSON processor.
    - [Installation Guide](https://jqlang.github.io/jq/download/)

You must also be authenticated with the GitHub CLI. If you haven't already, run:

```bash
gh auth login
```

## The Scripts

Here is a breakdown of each script located in the `scripts/` directory.

### 1. `get_commented_prs.sh`

- **Purpose**: This script searches the entire repository for pull requests that a specific user has commented on and prints a list of their corresponding numbers.
- **Usage**:
  ```bash
  sh ./scripts/get_commented_prs.sh <github_username>
  ```
- **Example**:
  ```bash
  sh ./scripts/get_commented_prs.sh joshuayoes
  ```
- **Output**: A simple list of pull request numbers, one per line.

<details>
<summary>View Script Content</summary>

```bash
#!/bin/bash

# A script to find all pull requests that a specific user has commented on
# within the current repository and output their numbers.

# --- Script ---
set -e # Exit immediately if a command exits with a non-zero status.
set -o pipefail # The return value of a pipeline is the status of the last command to exit with a non-zero status.

# Check for required tools
if ! command -v gh &> /dev/null; then
    echo "Error: The GitHub CLI ('gh') is not installed. Please install it to continue." >&2
    exit 1
fi

if ! command -v jq &> /dev/null; then
    echo "Error: 'jq' is not installed. Please install it to continue." >&2
    exit 1
fi

# Check for required arguments
if [ -z "$1" ]; then
  echo "Usage:   $0 <github_username>" >&2
  echo "Example: $0 joshuayoes" >&2
  exit 1
fi

USERNAME=$1

# Get the repository from the current git context
REPO=$(gh repo view --json owner,name -q '.owner.login + "/" + .name' 2>/dev/null)
if [ -z "$REPO" ]; then
  echo "Error: Could not determine GitHub repository from the current directory." >&2
  echo "Please run this script from inside a git repository." >&2
  exit 1
fi

echo "Fetching PR numbers commented on by '${USERNAME}' in repo '${REPO}'..."
echo "---"

# Search for pull requests in the repository commented on by the user.
# --limit 1000 sets a high limit to get all PRs, as the default is low.
# --json number gets only the PR number.
# jq '.[] | .number' extracts the number from each JSON object in the array.
gh search prs --repo "${REPO}" --commenter "${USERNAME}" --limit 1000 --json number | jq '.[] | .number'
```

</details>

### 2. `get_pr_comments.sh`

- **Purpose**: This script retrieves all comments made by a specific user on a single pull request. It is designed to find both general comments and line-level review comments.
- **Usage**:
  ```bash
  sh ./scripts/get_pr_comments.sh <pr_number> <github_username>
  ```
- **Example**:
  ```bash
  sh ./scripts/get_pr_comments.sh 1496 joshuayoes
  ```
- **Output**: A list of the user's comments from that specific PR.

<details>
<summary>View Script Content</summary>

```bash
#!/bin/bash

# A script to fetch all comments (review summaries and line-level comments)
# for a specific user from a GitHub pull request.

# --- Configuration ---
# Set to `true` to see verbose curl-like output from the gh api calls.
DEBUG=false

# --- Script ---
set -e # Exit immediately if a command exits with a non-zero status.
set -o pipefail # The return value of a pipeline is the status of the last command to exit with a non-zero status.

# Check for required tools
if ! command -v gh &> /dev/null; then
    echo "Error: The GitHub CLI ('gh') is not installed. Please install it to continue."
    exit 1
fi

if ! command -v jq &> /dev/null; then
    echo "Error: 'jq' is not installed. Please install it to continue."
    exit 1
fi

# Check for required arguments
if [ -z "$1" ] || [ -z "$2" ]; then
  echo "Usage:   $0 <pr_number> <github_username>"
  echo "Example: $0 1496 joshuayoes"
  exit 1
fi

PR_NUMBER=$1
USERNAME=$2

# Get the repository from the current git context
REPO=$(gh repo view --json owner,name -q '.owner.login + "/" + .name' 2>/dev/null)
if [ -z "$REPO" ]; then
  echo "Error: Could not determine GitHub repository from the current directory." >&2
  echo "Please run this script from inside a git repository." >&2
  exit 1
fi

if [ "$DEBUG" = true ]; then
    gh --debug=api ...
fi

echo "Fetching all comments for user '${USERNAME}' from PR #${PR_NUMBER} in repo '${REPO}'..."
echo "---"

# Fetch review summary comments (e.g., "LGTM!", "Nice work!")
# These are stored in the body of a review object.
gh api "repos/${REPO}/pulls/${PR_NUMBER}/reviews" -q '.[] | select(.user.login == "'"$USERNAME"'" and .body != null and .body != "") | .body'

# Fetch line-level review comments and general PR comments
# The GitHub API conveniently bundles these together in the `comments` endpoint for a pull request.
gh api "repos/${REPO}/pulls/${PR_NUMBER}/comments" -q '.[] | select(.user.login == "'"$USERNAME"'") | .body'
```

</details>

### 3. `archive_user_comments.sh` (The Main Script)

- **Purpose**: This is the orchestrator script. It automates the entire process by first calling `get_commented_prs.sh` to get the list of PRs, and then looping through each PR number to fetch the comments using `get_pr_comments.sh`. It compiles all the results into a single Markdown file.
- **Usage**:
  ```bash
  sh ./scripts/archive_user_comments.sh <github_username>
  ```

<details>
<summary>View Script Content</summary>

```bash
#!/bin/bash

# A script to archive all of a user's comments from all the PRs they have
# commented on in a repository into a single Markdown file.

# --- Configuration ---
# The name of the output file.
OUTPUT_FILE="pr_comments_archive.md"

# --- Script ---
set -e
set -o pipefail

# Check for required tools
if ! command -v gh &> /dev/null; then
    echo "Error: The GitHub CLI ('gh') is not installed. Please install it to continue." >&2
    exit 1
fi

if ! command -v jq &> /dev/null; then
    echo "Error: 'jq' is not installed. Please install it to continue." >&2
    exit 1
fi

# Check for required arguments
if [ -z "$1" ]; then
  echo "Usage:   $0 <github_username>" >&2
  echo "Example: $0 joshuayoes" >&2
  exit 1
fi

USERNAME=$1
SCRIPT_DIR=$(dirname "$0")

# Verify that the other scripts exist and are executable
if [ ! -x "${SCRIPT_DIR}/get_commented_prs.sh" ]; then
    echo "Error: Script 'get_commented_prs.sh' not found or not executable in '${SCRIPT_DIR}'." >&2
    exit 1
fi

if [ ! -x "${SCRIPT_DIR}/get_pr_comments.sh" ]; then
    echo "Error: Script 'get_pr_comments.sh' not found or not executable in '${SCRIPT_DIR}'." >&2
    exit 1
fi

# Get the repository from the current git context
REPO=$(gh repo view --json owner,name -q '.owner.login + "/" + .name' 2>/dev/null)
if [ -z "$REPO" ]; then
  echo "Error: Could not determine GitHub repository from the current directory." >&2
  echo "Please run this script from inside a git repository." >&2
  exit 1
fi

echo "Archiving all comments for user '${USERNAME}' in repo '${REPO}'..."

# Initialize or clear the output file
echo "# Pull Request Comment Archive for ${USERNAME} in ${REPO}" > "$OUTPUT_FILE"
echo "_Generated on $(date)_" >> "$OUTPUT_FILE"
echo "" >> "$OUTPUT_FILE"

# Get the list of PRs the user has commented on
echo "Step 1: Finding all PRs commented on by ${USERNAME}..."
PR_LIST=$("${SCRIPT_DIR}/get_commented_prs.sh" "$USERNAME" | sed '/---/d' | sed '/Fetching/d')


if [ -z "$PR_LIST" ]; then
    echo "No pull requests found with comments from ${USERNAME}."
    echo "No pull requests found with comments from ${USERNAME}." >> "$OUTPUT_FILE"
    exit 0
fi

echo "Step 2: Fetching comments for each PR and writing to '${OUTPUT_FILE}'..."

# Loop through each PR number and get the comments
while IFS= read -r PR_NUMBER; do
    echo " -> Fetching comments from PR #${PR_NUMBER}..."
    echo "---" >> "$OUTPUT_FILE"
    echo "## [Comments on PR #${PR_NUMBER}](https://github.com/${REPO}/pull/${PR_NUMBER})" >> "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"

    # Call the script to get comments for the current PR
    # Use a subshell to capture output; redirect stderr to stdout to catch potential errors from the script
    COMMENTS=$("${SCRIPT_DIR}/get_pr_comments.sh" "$PR_NUMBER" "$USERNAME" 2>&1 | sed '/---/d' | sed '/Fetching/d')

    if [ -n "$COMMENTS" ]; then
        # Format comments as a blockquote list
        echo "$COMMENTS" | sed 's/^/> /' >> "$OUTPUT_FILE"
    else
        echo "> No comments found for ${USERNAME} on this PR." >> "$OUTPUT_FILE"
    fi
    echo "" >> "$OUTPUT_FILE"

done <<< "$PR_LIST"

echo "---"
echo "Done! All comments have been archived in '${OUTPUT_FILE}'."
```

</details>

## How to Generate Your Comment Archive

This is the only command you need to run to get the final result.

1.  Navigate to the root of the repository in your terminal.
2.  Run the main archive script with your GitHub username:

    ```bash
    ./scripts/archive_user_comments.sh joshuayoes
    ```

### What Happens Next?

The script will:

1.  Tell you it's starting the process.
2.  Find all the PRs you've commented on.
3.  Loop through each PR and fetch your comments.
4.  Create a file named `pr_comments_archive.md` in the root of the repository.

The final output file will look something like this:

```markdown
# Pull Request Comment Archive for joshuayoes in infinitered/ignite

_Generated on Tue Jul 30 12:00:00 UTC 2024_

---

## [Comments on PR #2958](https://github.com/infinitered/ignite/pull/2958)

> A comment from PR 2958.
> Another comment from PR 2958.

---

## [Comments on PR #2817](https://github.com/infinitered/ignite/pull/2817)

> A comment from PR 2817.

...and so on.
```

You now have a complete, well-formatted archive of your comments for this repository.