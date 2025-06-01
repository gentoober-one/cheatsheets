# Git Cheatsheet

<!--
Git is a powerful, free, and open-source distributed version control system (DVCS) designed to handle everything from small to very large projects with speed and efficiency.
It's used for tracking changes in source code and other files, allowing multiple developers to collaborate.
This cheatsheet provides a quick reference for common Git commands and concepts.
-->

## Core Git Concepts (Brief Overview)

*   **Repository (Repo):** A collection of all files, history, and metadata for a project. Stored in a `.git` subdirectory within your project.
*   **Working Directory (or Working Tree):** The local checkout of project files that you directly edit.
*   **Staging Area (Index):** A file in the Git directory that stores information about what changes will go into your next commit. You use `git add` to move changes from the working directory to the staging area.
*   **Commit:** A snapshot of the staging area (changes to one or more files), saved to the repository's history. Each commit has a unique ID (SHA-1 hash), an author, a commit message, and points to one or more parent commits.
*   **Branch:** An independent line of development. It's essentially a movable pointer to a specific commit. The default branch is often named `main` or `master`.
*   **HEAD:** A special pointer that usually refers to the currently checked-out branch's tip (latest commit). If you check out a specific commit (detached HEAD), HEAD points to that commit.
*   **Tag:** A pointer to a specific commit, typically used to mark release versions (e.g., `v1.0`).
*   **Remote:** A version of your repository hosted on a server (e.g., on GitHub, GitLab, Bitbucket). Your local repository can be connected to one or more remotes.
*   **Local Repository:** Your copy of the repository on your machine.

## Initial Setup & Configuration

### Creating a New Repository
<!-- You can initialize a repository in a new directory or an existing one. -->

#### Method 1: Create a new directory and then initialize
```bash
# Replace <repository-name> with your desired directory name.
mkdir <repository-name>
cd <repository-name>
git init
# This creates a .git subdirectory in your project directory, which contains all Git metadata.
```

#### Method 2: Initialize directly in an existing directory
```bash
# Navigate to your existing project directory.
# cd /path/to/your/project
git init
```
<!-- Or, initialize and create the directory at the same time (less common for `git init`): -->
<!-- `git init <repository-name>` creates a directory named <repository-name> and initializes a repo inside. -->

### Cloning an Existing Repository
<!-- Creates a local copy of a remote repository. -->
```bash
# git clone <repository-url>
#   <repository-url>: The URL of the remote repository (e.g., from GitHub, GitLab).
# This automatically creates a directory named after the repository.
git clone https://github.com/user/repository.git

# Clone into a specific directory name:
# git clone <repository-url> <desired-local-directory-name>
```

## Git Configuration (User Setup)
<!-- Configure your Git identity. This information is used in your commits. -->
<!-- These settings are typically global, applied to all your Git repositories. -->

### Global Git Configuration
<!-- These settings are stored in `~/.gitconfig` (or `~/.config/git/config`). -->
```bash
# Set your user name (this will appear in the commit history)
git config --global user.name "Your Name"

# Set your email address (this will also appear in commit history)
git config --global user.email "you@example.com"

# Set the default branch name for new repositories created with `git init`
# Common choices are "main" (recommended) or "master".
git config --global init.defaultBranch main

# Set your preferred text editor for commit messages, interactive rebases, etc.
# git config --global core.editor "nano" # Example: use nano
# git config --global core.editor "vim"
# git config --global core.editor "code --wait" # Example: use VS Code (ensure it's in your PATH)

# Enable colorful output for Git commands (makes it easier to read diffs, status, etc.)
git config --global color.ui auto
```

### Repository-Specific Configuration
<!-- To set configuration options for a specific repository, navigate into the repo's directory
     and run `git config` without the `--global` flag. These settings are stored in `.git/config`. -->
```bash
# cd /path/to/your/repo
# git config user.name "RepositorySpecificName"
# git config user.email "repo-specific-email@example.com"
```

### Viewing Configuration
```bash
# List all Git configuration settings (local, global, and system, in that order of precedence)
git config --list

# View global configuration only
# git config --global --list

# View repository-specific (local) configuration only
# git config --local --list

# Get a specific configuration value
# git config user.name # Shows the configured user name
```

## Creating & Cloning Repositories

### Initializing a New Repository
```bash
# Method 1: Create a new directory, enter it, and then initialize a Git repository
mkdir my-new-project
cd my-new-project
git init # Creates a .git subdirectory

# Method 2: Initialize an existing project directory as a Git repository
# cd /path/to/existing/project
# git init

# Method 3: Initialize and create the directory at the same time
# git init my-new-project-dir # Creates 'my-new-project-dir' and initializes it
```

### Cloning an Existing Repository
<!-- Creates a local copy of a remote repository, including all history and branches. -->
```bash
# git clone <repository-url>
# This automatically creates a directory named after the repository.
git clone https://github.com/someuser/some-repository.git

# Clone into a specific directory name:
git clone https://github.com/someuser/some-repository.git my-custom-app
```

## Basic Git Workflow (Daily Use)

### Checking Status
<!-- Shows the status of changes as untracked, modified, or staged. -->
```bash
git status
```

### Staging Files (`git add`)
<!-- Adds file contents from the working directory to the staging area (index) for inclusion in the next commit. -->

#### Stage All Changes in Current Directory and Subdirectories
```bash
git add .
# '.' refers to the current directory. This stages new files and modifications to existing files.
```

#### Stage a Specific File or Directory
```bash
git add path/to/your/file.txt
git add path/to/your_directory/
```

#### Stage All Modified and Deleted Files (but not new untracked files)
```bash
git add -u
# Useful for staging all changes to already tracked files.
```

#### Stage All Files (Modified, Deleted, and New Untracked files)
```bash
git add -A
# Often equivalent to `git add .` in modern Git versions for the current directory.
```

#### Stage Specific Parts of a File (Interactive Staging)
```bash
# git add -p <file-path>
#   -p or --patch: Allows you to interactively review and choose which changes (hunks) within a file to stage.
#   Useful for breaking down large changes into smaller, logical commits.
git add -p path/to/your/file.txt
```

### Unstaging Files (`git restore --staged` or `git reset`)
<!-- Removes a file from the staging area, but its modifications in the working directory are kept. -->
```bash
# Modern way (Git 2.23+):
git restore --staged path/to/your/file.txt

# To unstage everything that is currently staged:
# git restore --staged .

# Older way (still common and functional):
# git reset HEAD path/to/your/file.txt # Unstages a specific file
# git reset # Unstages all currently staged files
```

### Viewing Changes (`git diff`)
<!-- Shows differences between various states (working directory, staging area, commits). -->
```bash
# Show unstaged changes (differences between working directory and staging area)
git diff

# Show staged changes (differences between staging area and last commit/HEAD)
git diff --staged
# Or: git diff --cached

# Show all changes in tracked files since last commit (both staged and unstaged combined)
git diff HEAD

# Show differences between two commits
# git diff <commit-hash-1> <commit-hash-2>

# Show differences between a branch and current HEAD
# git diff <branch-name> # Compares tip of <branch-name> with current working directory
# git diff <branch-name> HEAD # Compares tip of <branch-name> with current HEAD commit
```

### Committing Changes (`git commit`)
<!-- Records snapshots of the staging area (changes added with `git add`) to the repository's history. -->

#### Commit with a Short Message
```bash
# git commit -m "Your concise commit message"
#   -m: Allows you to provide the commit message inline.
git commit -m "Fix: Corrected login bug #123"
```

#### Commit with a More Detailed Message (Opens Editor)
```bash
git commit
# This opens your configured text editor to write a longer commit message.
# Good practice:
#   - Short summary line (max 50 characters).
#   - Followed by a blank line.
#   - Then a more detailed explanation, wrapped at ~72 characters.
```

#### Stage All Tracked (Modified/Deleted) Files and Commit
```bash
# git commit -a -m "Your message"
#   -a: Automatically stage files that have been modified or deleted (does NOT stage new untracked files).
# Use with caution; it's generally better to `git add` explicitly to ensure you know what's being committed.
git commit -a -m "Refactor: Updated user authentication module"
```

#### Amend the Last Commit
<!-- Modify the most recent commit. Useful for fixing typos in the message or adding forgotten files. -->
<!-- WARNING: Avoid amending commits that have already been pushed to a shared remote, as it rewrites history
     and can cause problems for collaborators. Safe for local commits not yet pushed. -->
```bash
# First, add any forgotten files to staging area if needed:
# git add forgotten-file.txt
git commit --amend -m "New, corrected commit message for previous commit"
# Or, to just change the message without adding new files:
# git commit --amend
# To add currently staged changes and amend the previous commit, keeping the old message:
# git commit --amend --no-edit
```

### Ignoring Files (`.gitignore`)
<!-- Specifies intentionally untracked files that Git should ignore.
     Ignored files won't show up in `git status` (unless they were previously tracked) and won't be staged with `git add .`. -->
<!-- Create a file named `.gitignore` in your repository's root directory (or subdirectories for specific rules). -->
```
# Example .gitignore content:

# Comments start with a hash

# Ignore build output directories
build/
dist/
out/

# Ignore specific file types
*.log
*.tmp
*.o
*.class

# Ignore OS-generated files
.DS_Store
Thumbs.db

# Ignore dependency directories (often managed by package managers)
node_modules/
vendor/
target/ # For Java/Maven/Rust

# Ignore sensitive information (should ideally never be committed)
.env
secrets.json
*.pem
```
<!-- After creating or updating `.gitignore`, add it to your repository:
git add .gitignore
git commit -m "Add or update .gitignore"
-->

### Viewing Commit History (`git log`)
<!-- Shows the commit history of the current branch. -->
```bash
git log

# Common options:
# --oneline: Show each commit on a single line (hash and commit message subject).
# --graph: Display an ASCII graph of the branch structure.
# --decorate: Show branch names, tags, and HEAD pointers.
# --all: Show history for all branches, not just the current one.
# --patch or -p: Show the changes (diff) introduced in each commit.
# --stat: Show summary of changes (files changed, insertions/deletions).
# --author="<pattern>": Filter commits by author.
# --grep="<pattern>": Filter commits by commit message content.
# --since="2 weeks ago" or --until="2023-01-01": Filter by date.
# <path/to/file>: Show log only for a specific file or directory.
# Example: View a decorated, graphed, one-line history for all branches
git log --oneline --graph --decorate --all
```

## Branching

<!--
Best Practice: Work on Feature Branches
<!-- Branches are used to develop features, fix bugs, or experiment in isolation from the main codebase (e.g., `main` branch).
     This keeps the main line of development clean and stable. -->
<!-- Best Practice: Work on Feature Branches. It's generally recommended to do development work on separate "feature" branches
     rather than directly on the `main` (or `master`) branch. This keeps `main` stable and allows for code review
     via Pull/Merge Requests before changes are integrated. -->

### Listing Branches
```bash
# List all local branches. The current branch is marked with an asterisk (*).
git branch

# List all branches (local and remote-tracking)
git branch -a

# List branches with more details (e.g., last commit hash and message, tracking info)
git branch -vv
```

### Creating a New Branch
```bash
# Create a new branch based on the current commit (but you stay on the current branch)
# git branch <new-branch-name>
git branch feature/new-user-interface

# Create a new branch from a specific commit or another branch
# git branch <new-branch-name> <start-point>
# git branch hotfix/critical-bug-fix main # Create 'hotfix/critical-bug-fix' based on the 'main' branch
```

### Switching Branches
```bash
# Modern way (Git 2.23+): `git switch` (preferred for just switching branches)
# git switch <branch-name>
git switch feature/new-user-interface # Switch to the 'feature/new-user-interface' branch
git switch main                       # Switch back to 'main'

# Older way (still widely used): `git checkout`
# git checkout <branch-name>
# git checkout feature/new-user-interface
```

### Creating and Switching to a New Branch (in one step)
```bash
# Modern way (Git 2.23+):
# git switch -c <new-branch-name>
git switch -c feature/user-registration-flow

# Create from a specific start point and switch
# git switch -c <new-branch-name> <start-point-branch-or-commit>
# git switch -c hotfix/quick-patch main

# Older way: `git checkout -b`
# git checkout -b <new-branch-name>
# git checkout -b feature/user-settings
```

### Deleting a Local Branch
```bash
# Delete a branch only if it has been fully merged into the current branch or an upstream branch.
# This is a safety measure to prevent accidental loss of unmerged changes.
# git branch -d <branch-name>
git branch -d feature/old-login-page

# Force delete a branch, even if it has not been fully merged.
# Use with caution, as this can lead to loss of unmerged work if you're not careful.
# git branch -D <branch-name>
git branch -D feature/abandoned-experimental-work
```

### Renaming a Branch
```bash
# Rename the current local branch
# git branch -m <new-name>
git branch -m feature/improved-profile-page # If currently on the branch to be renamed

# Rename another local branch (when you are on a different branch)
# git branch -m <old-name> <new-name>
# git branch -m old-feature-name new-feature-name-final
```

## Merging
<!-- Merging combines changes from different branches into one.
     Typically, you merge a feature branch back into your main development branch (e.g., `main`). -->

### Basic Merge
```bash
# 1. Switch to the branch you want to merge changes INTO (this is the target branch, e.g., main).
git switch main

# 2. Merge the desired branch (this is the source branch, e.g., feature/user-profile) into the current branch.
# git merge <branch-to-merge-from>
git merge feature/user-profile
# This creates a new "merge commit" by default if the target branch has new commits since the feature branch was created.
# If the target branch has no new commits (feature branch is directly ahead), it will be a "fast-forward" merge (no merge commit, just moves the pointer).
```

### Handling Merge Conflicts
<!-- Conflicts occur if changes from both branches affect the same lines in the same file, or if one branch deletes a file that another modifies.
     Git will pause the merge and mark the conflicted files. -->
```bash
# 1. Identify conflicted files: `git status` will show them.
# 2. Open the conflicted file(s) in your editor. You'll see conflict markers:
#    <<<<<<< HEAD
#    (changes from your current branch - e.g., main)
#    =======
#    (changes from the branch being merged - e.g., feature/user-profile)
#    >>>>>>> feature/user-profile
# 3. Manually edit the file to resolve the differences: choose which parts to keep, or combine them. Remove the conflict markers.
# 4. Stage the resolved file(s):
git add path/to/resolved/conflicted-file.txt
# 5. Continue the merge. Git usually prepares a merge commit message, which you can edit.
git commit
# Or, if you want to stop the merge and go back to the state before merging:
git merge --abort
```

### Merge Strategies (Brief Note)
<!-- Git has various merge strategies. The default (`ort` or `recursive`) is usually fine. -->
<!-- `--no-ff`: Create a merge commit even if it's a fast-forward. Useful for keeping a record of feature merges. -->
<!-- `git merge --no-ff feature/some-feature` -->
<!-- `--squash`: Combine all commits from the feature branch into a single commit on the target branch. The original commits on the feature branch are not part of the history of the target branch. -->
<!-- `git merge --squash feature/another-feature` (followed by `git commit`) -->

## Rebasing (`git rebase`)
<!-- Rebasing reapplies commits from one branch on top of another branch, effectively rewriting commit history.
     It's useful for maintaining a linear project history, especially for feature branches before merging them into `main`. -->
<!-- WARNING: Do NOT rebase commits that have already been pushed to a shared/public remote repository,
     unless you are coordinating with your team and understand the implications. Rebasing rewrites history,
     which can cause significant problems for collaborators if they have based their work on the old history.
     It's generally safe for local feature branches that haven't been shared yet. -->

### Basic Rebase
```bash
# 1. Ensure your main/base branch is up-to-date:
git switch main
git pull origin main

# 2. Switch to the feature branch you want to rebase.
git switch feature/user-profile

# 3. Rebase it onto the target base branch (e.g., main).
# git rebase <base-branch-name>
git rebase main
# This takes commits from 'feature/user-profile' that are not in 'main'
# and reapplies them, one by one, on top of the latest commit of 'main'.
# The commit hashes on your feature branch will change.

# If conflicts occur during rebase:
#   a. Git will stop at the conflicting commit. Resolve conflicts in the marked files.
#   b. Stage the resolved files: `git add <conflicted-file-path>`
#   c. Continue the rebase: `git rebase --continue`
#   Or, to skip the problematic commit entirely (use with caution): `git rebase --skip`
#   Or, to abort the rebase and return to the state before it started: `git rebase --abort`

# After a successful rebase, your feature branch is updated and has a linear history with 'main'.
# If this feature branch was already pushed to a remote, you'll need to force push it:
# git push origin feature/user-profile --force-with-lease
# (Use --force-with-lease for safety over --force)

# Then, back on 'main', merging the rebased feature branch will typically be a fast-forward merge:
# git switch main
# git merge feature/user-profile # Should be fast-forward
```

### Interactive Rebase (`git rebase -i`)
<!-- Allows you to modify commits as they are rebased: reorder, squash (combine), edit messages, drop (delete), etc. -->
```bash
# Rebase the last N commits interactively (e.g., last 3 commits on current branch):
git rebase -i HEAD~3

# Rebase interactively onto another branch:
# git rebase -i main # If on a feature branch, rebase onto main interactively

# This opens an editor with a list of commits and actions (pick, squash, edit, reword, drop, etc.).
# Follow the instructions provided in the editor comments.
# - `pick`: Use commit as is.
# - `reword` (or `r`): Use commit, but edit the commit message.
# - `edit` (or `e`): Use commit, but stop for amending (e.g., to change content, split commit).
# - `squash` (or `s`): Combine this commit's changes into the previous commit, and meld commit messages.
# - `fixup` (or `f`): Like squash, but discard this commit's message.
# - `drop` (or `d`): Remove commit.
# Save and close the editor. Git will then process the commits according to your instructions.
# Resolve conflicts if they occur during this process.
```

## Working with Remote Repositories

### Viewing Remotes
```bash
# List the shortnames of your remotes (e.g., "origin").
git remote

# List remotes with their URLs (for fetch and push).
git remote -v
```

### Adding a Remote Repository
<!-- Connects your local repository to a remote server. -->
```bash
# git remote add <remote-name> <repository-url>
#   <remote-name>: A shorthand name for the remote (commonly "origin").
#   <repository-url>: The URL of the remote repository.
# Example for GitHub using HTTPS:
git remote add origin https://github.com/your-username/your-repository-name.git
# Example for GitHub using SSH (often preferred for authenticated access):
# git remote add origin git@github.com:your-username/your-repository-name.git
```

### Renaming a Remote
```bash
# git remote rename <old-name> <new-name>
git remote rename origin upstream
```

### Removing a Remote
```bash
# git remote remove <remote-name>
# Or: git remote rm <remote-name>
git remote remove old-remote-name
```

### Pushing Changes to a Remote Repository
<!-- Uploads local branch commits to the remote repository. -->

#### Push a Branch (and set upstream tracking for the first time)
```bash
# git push -u <remote-name> <local-branch-name>:<remote-branch-name>
#   -u or --set-upstream: Sets up tracking between your local branch and the remote branch.
#                         After this, you can often just use `git push` from that branch.
# If local and remote branch names are the same:
git push -u origin main # Pushes local 'main' to 'origin/main' and sets up tracking

# Push a new local feature branch to remote and set tracking:
git push -u origin feature/new-login-page
```

#### Push (if upstream tracking is already set)
```bash
# If your local branch is already tracking a remote branch (e.g., after `push -u` or cloning):
git push
```

#### Force Pushing (Use with EXTREME CAUTION)
<!-- Overwrites the remote branch with your local branch. Can discard remote commits if others have pushed. -->
<!-- WARNING: Avoid force pushing to shared branches (like main, develop) unless you are coordinating with your team
     and understand the implications. It's generally safer for feature branches that only you work on, especially after a rebase. -->
```bash
# git push --force <remote-name> <branch-name>
# A safer alternative that only forces if the remote branch hasn't changed unexpectedly since your last fetch:
git push --force-with-lease <remote-name> <branch-name>
```

#### Deleting a Remote Branch
```bash
# git push <remote-name> --delete <branch-name>
git push origin --delete feature/old-or-merged-feature
# Older syntax (still works but less intuitive):
# git push origin :feature/old-or-merged-feature
```

### Fetching Changes from a Remote
<!-- Downloads new data (commits, branches, tags) from a remote repository into your local repository.
     It updates your remote-tracking branches (e.g., `origin/main`) but does NOT integrate these changes
     into your local working branches (e.g., `main`). -->
```bash
# git fetch <remote-name>
# Example: Fetch all updates from 'origin'.
git fetch origin

# Fetch from all remotes:
# git fetch --all

# Prune (delete) local remote-tracking branches that no longer exist on the remote:
git fetch --prune origin
# Or set this to happen automatically: git config --global fetch.prune true
```

### Pulling Changes from a Remote
<!-- Fetches changes from a remote repository AND immediately tries to merge them into your current local branch.
     `git pull <remote> <branch>` is roughly equivalent to `git fetch <remote> <branch>` followed by `git merge FETCH_HEAD` (or merging the tracked remote branch). -->
```bash
# git pull <remote-name> <remote-branch-name>
# Example: Pull changes from 'origin's main branch and merge into your current local branch.
git pull origin main

# If your local branch is tracking a remote branch (common after `git clone` or `git push -u`):
git pull
# This pulls from the tracked upstream branch and merges into the current local branch.

# To pull and rebase your local unpushed commits on top of the fetched changes (instead of creating a merge commit):
# This keeps a more linear history for your local commits.
git pull --rebase <remote-name> <remote-branch-name>
# Example: git pull --rebase origin main
# Or if tracking is set up:
# git pull --rebase
```

## Undoing Changes & Recovering

<!-- Commands to undo mistakes, discard changes, or recover lost work. Use with caution, especially commands that rewrite history. -->

### Discarding Changes in Working Directory
<!-- Reverts file(s) in your working directory to their state at the last commit (HEAD) or from the staging area. -->
<!-- WARNING: Any local modifications to the file(s) that haven't been committed (and are not staged) will be PERMANENTLY LOST. -->
```bash
# Modern way (Git 2.23+): Discard changes in a specific file in the working directory.
git restore path/to/your/file.txt

# Discard all changes in the working directory to all tracked files in current directory and subdirectories:
# git restore . # Use with extreme caution.

# Older way: `git checkout -- <file-path>`
# git checkout -- path/to/your/file.txt
# git checkout -- . # Discard all changes to tracked files in current dir and subdirs. Also very dangerous.
```

### Unstaging a File (from Staging Area)
<!-- Removes a file from the staging area, but its modifications in the working directory are kept.
     The file moves from "staged" back to "modified" (if it was modified before staging). -->
```bash
# Modern way (Git 2.23+):
git restore --staged path/to/your/file.txt

# To unstage everything that is currently staged:
# git restore --staged .

# Older way (still common and functional):
# git reset HEAD path/to/your/file.txt # Unstages a specific file
# git reset # Unstages all currently staged files
```

### Reverting a Commit (Safe Undo for Shared History)
<!-- Creates a new commit that undoes the changes introduced by a specified previous commit.
     This is a safe way to undo changes on public/shared branches as it doesn't rewrite existing history; it appends a new commit. -->
```bash
# git revert <commit-hash>
#   <commit-hash>: The hash of the commit you want to undo.
# This will open an editor for the revert commit message. Describe why you are reverting.
git revert HEAD # Reverts the changes of the very last commit on the current branch
# git revert <specific-commit-id> # Reverts a specific commit

# If reverting a merge commit, you might need to specify which parent to revert to:
# git revert -m 1 <merge-commit-hash> # -m 1 usually means revert to the state of the first parent
```

### Resetting Commits (Rewrites History - Use with Caution Locally)
<!-- Moves the current branch pointer (HEAD) back to a specified commit, potentially altering history. -->
<!-- WARNING: Use with extreme caution. Avoid using `git reset` (especially `--hard`) on commits that have been pushed to a shared remote,
     as it rewrites history and can cause major issues for collaborators.
     If you need to undo a reset (and haven't run `git gc`), `git reflog` can help find "lost" commits. -->

#### Soft Reset (Move HEAD, keep changes from "undone" commits in Staging Area)
```bash
# git reset --soft <commit-hash_or_reference>
# The branch pointer moves to <commit-hash>. Changes from commits that came after <commit-hash> on this branch
# are now considered "changes to be committed" (i.e., they are in the staging area).
# Useful for redoing a series of commits or squashing them into one.
# Example: Go back one commit from HEAD; changes from that commit are now staged.
git reset --soft HEAD~1
# You can then `git commit` again with a new message or combine changes.
```

#### Mixed Reset (Default - Move HEAD, keep changes in Working Directory, Unstage)
```bash
# git reset <commit-hash_or_reference>
# Or: git reset --mixed <commit-hash_or_reference> (this is the default behavior of `git reset`)
# The branch pointer moves to <commit-hash>. Changes from commits after <commit-hash> are kept
# in the working directory but are unstaged.
# Example: Go back one commit from HEAD; changes from that commit are in your working directory as unstaged changes.
git reset HEAD~1
```

#### Hard Reset (DANGEROUS - Move HEAD, Discard All Changes)
<!-- Discards all changes in the staging area AND working directory since <commit-hash_or_reference>.
     Any uncommitted work and any commits on this branch after <commit-hash_or_reference> will be PERMANENTLY LOST locally
     (recovery might be possible via `git reflog` if done quickly, but consider them gone). -->
```bash
# git reset --hard <commit-hash_or_reference>
# Example: Discard all local changes and commits since origin/main, making local main identical to remote.
# git fetch origin
# git reset --hard origin/main # DANGEROUS if you have local unpushed commits on main.

# Example: Go back one commit, discarding all changes made in HEAD (the last commit).
# git reset --hard HEAD~1
```

### Cleaning Untracked Files from Working Directory
<!-- Removes untracked files (files not ignored and not yet added to Git). -->
<!-- WARNING: This permanently deletes files. Use with extreme caution. -->
```bash
# Dry run: Show what would be deleted, without actually deleting.
git clean -n
#   -n or --dry-run: Show what would be done.
#   -d: Also consider untracked directories (by default, only files).
#   -f or --force: Required to actually delete files/directories.
#   -x: Also remove files ignored by .gitignore (even more dangerous).
#   -X: Only remove files ignored by .gitignore.

# Example dry run, including directories:
git clean -n -d

# Actually delete untracked files:
# git clean -f

# Delete untracked files AND directories:
# git clean -f -d

# Delete untracked files, directories, AND ignored files (very thorough cleanup):
# git clean -f -d -x
```

### Using `git reflog` (Your Safety Net)
<!-- Shows a log of where HEAD and branch tips have been. It records updates to the tip of branches and other references in the local repository.
     Crucial for recovering "lost" commits or undoing mistakes, especially after operations that rewrite history like `git reset --hard` or a rebase. -->
```bash
git reflog
# Each entry shows a reference (e.g., HEAD@{0}, main@{2}) and the commit hash it pointed to, along with the action that moved it.
# You can use these references (like HEAD@{N}) in other commands like `git reset` or `git checkout`.

# Example: If you accidentally did `git reset --hard HEAD~3` and lost three commits:
# 1. Run `git reflog`. Find the entry before the hard reset (e.g., HEAD@{4} might be where HEAD was).
# 2. Reset your current branch back to that commit:
#    git reset --hard HEAD@{4} # Or the specific commit hash from the reflog.
```

## Stashing Changes (`git stash`)
<!-- Temporarily shelves (stashes) uncommitted changes (staged and unstaged modifications to tracked files)
     in your working directory. This allows you to quickly switch contexts (e.g., to another branch)
     without committing incomplete work, and then re-apply the changes later. -->
```bash
# Stash current uncommitted changes.
# This includes changes in the staging area and working directory for tracked files.
# Untracked files are not stashed by default.
git stash
# Or, provide a message for the stash for easier identification:
git stash push -m "WIP: Implementing user login form"

# Stash including untracked files:
# git stash -u
# Or: git stash --include-untracked

# Stash all files (untracked, ignored, and tracked):
# git stash --all

# List all stashes (stashes are stored in a stack-like structure)
git stash list
# Output might look like:
# stash@{0}: WIP on main: abc1234 Last commit message
# stash@{1}: On feature/login: My temporary work on feature X

# Apply the most recent stash (stash@{0})
# The changes are reapplied to your working directory. The stash itself is kept in the stash list.
git stash apply
# Apply a specific stash from the list:
# git stash apply stash@{2}

# Apply the most recent stash and remove it from the stash list (pop)
# If there are conflicts, the stash is NOT dropped automatically; resolve conflicts then `git stash drop`.
git stash pop
# Pop a specific stash:
# git stash pop stash@{1}

# Show changes recorded in a stash as a diff
git stash show stash@{0} # Shows summary of changes for the latest stash
# Show full patch (diff) for the latest stash:
git stash show -p stash@{0}

# Remove the most recent stash from the list
git stash drop
# Remove a specific stash:
git stash drop stash@{1}

# Remove all stashes (use with caution)
git stash clear

# Create a branch from a stash (useful if stashed work becomes more significant)
# git stash branch <new-branch-name> [stash@{N}]
# git stash branch feature/from-stash stash@{0}
```

## Tagging (`git tag`)
<!-- Tags are used to mark specific points in the repository's history as important, typically for releases (e.g., v1.0.0). -->

### Listing Tags
```bash
git tag
# Filter tags using a pattern (e.g., all v1.* tags)
# git tag -l "v1.*"
```

### Creating Tags

#### Lightweight Tag
<!-- A simple pointer to a specific commit. It's like a branch that doesn't move.
     It doesn't store extra information like tagger, date, or message. -->
```bash
# git tag <tagname> [commit-hash]
# If [commit-hash] is omitted, tags the current commit (HEAD).
git tag v1.0.0-lw # Tags the current commit

# Tag a specific past commit:
# git tag v0.9.0 abc123fg
```

#### Annotated Tag (Recommended for releases)
<!-- Stores extra metadata: tagger name, email, date, and a tagging message.
     Annotated tags are full objects in the Git database and can be GPG-signed. -->
```bash
# git tag -a <tagname> -m "Your tag message" [commit-hash]
# If -m is omitted, Git opens your editor to write the tag message.
git tag -a v1.0.1 -m "Version 1.0.1 release with critical bug fixes"

# GPG-sign an annotated tag:
# git tag -s <tagname> -m "Signed release message"
# (Requires GPG to be configured)
```

### Viewing Tag Details
```bash
# For annotated tags, `git show` displays tag information (tagger, date, message) and the commit details.
# For lightweight tags, it just shows the commit details.
git show v1.0.1
```

### Pushing Tags to Remote
<!-- By default, `git push` does not push tags. You need to push them explicitly. -->
```bash
# Push a specific tag to a remote (e.g., origin)
git push origin v1.0.1

# Push all local tags to the remote (that are not yet on the remote)
git push origin --tags
```

### Deleting Tags
```bash
# Delete a local tag
git tag -d v1.0.0-beta

# Delete a remote tag (this involves pushing a "null" ref for the tag to the remote)
# git push <remote-name> :refs/tags/<tagname> (older syntax)
# Or, more modern and intuitive:
git push origin --delete v1.0.0-beta
```

### Checking Out a Tag
<!-- You can check out a tag to view the repository state at that point.
     This puts you in a "detached HEAD" state. -->
```bash
# git checkout <tagname>
# git checkout v1.0.1
# If you want to make changes starting from this tag, create a new branch:
# git switch -c new-branch-from-tag v1.0.1
```

## Inspecting History & Diffs (Advanced `git log` & More)

<!-- Commands to explore the repository's history, view changes between commits, and see who changed what. -->

### `git log` (Recap and More Options)
<!-- Shows the commit history. Many options to customize output. -->
```bash
# Basic log
git log

# One-line summary per commit
git log --oneline

# With graph, decoration (branch/tag names), and all branches
git log --oneline --graph --decorate --all

# Show patch (diff) for each commit
git log -p
# Show patch for a specific file
# git log -p path/to/file.txt

# Show statistics (files changed, insertions/deletions) for each commit
git log --stat

# Filter by author
# git log --author="John Doe"

# Filter by committer
# git log --committer="Jane Doe"

# Filter by commit message content (grep)
# git log --grep="Fix bug #123"

# Filter by date range
# git log --since="2 weeks ago"
# git log --until="2023-01-01"
# git log --since="2023-01-01" --until="2023-01-31"

# Filter by specific file or directory path
# git log -- path/to/your/file.txt
# git log -- path/to/your/directory/

# Show a specific number of commits
# git log -5 # Shows the last 5 commits

# Pretty formatting options
# git log --pretty=format:"%h - %an, %ar : %s"
#   %h: abbreviated commit hash, %an: author name, %ar: author date relative, %s: subject
```

### `git show`
<!-- Shows various types of objects (blobs for file content, trees for directories, tags, commits).
     For commits, it shows the log message and a diff of the changes introduced by that commit. -->
```bash
git show <commit-hash_or_tag_or_branch_or_HEAD>
# Example: Show details of the commit two before HEAD
# git show HEAD~2
# Example: Show details of tag v1.0.1 and the commit it points to
# git show v1.0.1
# Example: Show the state of a file at a specific commit
# git show <commit-hash>:path/to/file.txt
```

### `git diff` (Recap for Comparing Commits/Branches)
<!-- While also used for working directory/staging area, `git diff` is powerful for comparing history. -->
```bash
# Show differences between two arbitrary commits
git diff <commit-hash-1> <commit-hash-2>

# Show differences between the tips of two branches
git diff main feature/new-login

# Show changes on `feature/new-login` that are not on `main`
git diff main...feature/new-login # Note the three dots

# Show changes for a specific file between two commits
# git diff <commit-hash-1>:<file1> <commit-hash-2>:<file2> # If paths are different
# git diff <commit-hash-1> <commit-hash-2> -- path/to/file.txt # If path is same
```

### `git blame`
<!-- Shows who last modified each line of a file, and in which commit. Useful for tracking down when a specific change was introduced and by whom. -->
```bash
# git blame <file-path>
git blame README.md
# Options:
#   -L <start_line>,<end_line>: Show blame only for the specified line range.
#   -e: Show email address of the author instead of the name.
#   -C: Detect moved or copied lines within the same file (or from other files if specified multiple times).
```

### `git shortlog`
<!-- Summarizes `git log` output, grouping commits by author and displaying the first line of each commit message.
     Useful for generating changelogs or seeing contributions per author. -->
```bash
git shortlog
# Options:
#   -s or --summary: Suppress commit descriptions, show only commit counts per author.
#   -n or --numbered: Sort output according to the number of commits per author (descending) instead of author name.
#   -e or --email: Show email address of the author.
# Example: Summary of commit counts by author, sorted by count, across all branches
git shortlog -sn --all
```

## Other Useful Git Commands

### `git bisect` (Finding Faulty Commits)
<!-- Helps find the commit that introduced a bug by performing a binary search through the commit history. -->
```bash
# Start a bisect session
git bisect start

# Mark a known "bad" commit (e.g., current HEAD where bug exists)
git bisect bad # Or git bisect bad <commit-hash>

# Mark a known "good" commit (e.g., a tag or older commit where bug didn't exist)
git bisect good v1.0.0 # Or git bisect good <commit-hash>

# Git will then check out a commit in the middle. Test for the bug.
# If bug is present:
git bisect bad
# If bug is NOT present:
git bisect good

# Repeat until Git identifies the first bad commit.
# To end the bisect session and return to your original branch:
git bisect reset
```

### `git submodule` (Managing External Repositories)
<!-- Allows you to include and manage another Git repository as a subdirectory within your main repository. -->
```bash
# Add a new submodule
# git submodule add <repository-url> path/to/submodule
# git submodule add https://github.com/user/library.git extern/my-library

# Clone a repository with submodules (or initialize them after cloning)
# git clone --recurse-submodules <repository-url>
# If already cloned:
# git submodule init # Initialize submodule configuration
# git submodule update # Fetch and checkout appropriate commit for submodules

# Update submodules to their latest commit (as specified by the submodule's default branch)
# git submodule update --remote path/to/submodule

# To pull changes within a submodule and then commit that new state in the parent repo:
# cd path/to/submodule
# git pull origin main (or appropriate branch)
# cd ../.. (back to parent repo)
# git add path/to/submodule
# git commit -m "Update submodule my-library"
```

### `git cherry-pick` (Applying a Specific Commit)
<!-- Applies the changes introduced by an existing commit from another branch to the current branch.
     It creates a new commit on the current branch with these changes. -->
```bash
# git cherry-pick <commit-hash>
# Example: Apply commit 'abc123xyz' from 'feature-branch' onto 'main' (if currently on main)
git cherry-pick abc123xyz
# Useful for picking specific bug fixes without merging an entire feature branch.
# Can cause conflicts if the cherry-picked changes overlap with current branch changes.
```