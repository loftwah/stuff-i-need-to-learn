# Git

Hey, we use this all the time so it never hurts to have a quick reference and to practice.

## Force Reset to Remote Branch

When you want to completely reset your local branch to match the remote (like a `git pull --force`):

```bash
git fetch origin
git reset --hard origin/main
```

**What this does:**

- `git fetch origin`: Gets the latest from remote without merging
- `git reset --hard origin/main`: Discards all local changes and history divergence, setting your current branch to exactly match origin/main

**⚠️ Warning:** This will destroy all uncommitted changes and overwrite merge conflicts.

## Handling Merge Conflicts

### Abort a Merge

If you don't want the merge at all:

```bash
git merge --abort
```

### Keep Only Main Version During Conflicts

To accept the current branch (main) version for all conflicted files:

```bash
git restore --source=HEAD --staged --worktree .
git commit -m "Keep main version during merge"
```

### Resolve Specific Files

Keep main's version for specific files:

```bash
git checkout --ours path/to/file
# Or for all files:
git checkout --ours .
git add .
git commit -m "Resolved conflicts using ours strategy"
```

## Keeping Feature Branches Up-to-Date

When working on a long-lived branch and main has moved forward, you have two options:

### Option 1: Merge (Safe, explicit, but messy history)

```bash
git checkout feature-branch
git fetch origin
git merge origin/main
```

**Pros:** Safe, keeps full history visible
**Cons:** Creates merge commits, messier history

### Option 2: Rebase (Clean, linear history - preferred for feature branches)

```bash
git checkout feature-branch
git fetch origin
git rebase origin/main
```

**Pros:** Clean linear history, better for PRs
**Cons:** Rewrites history (riskier if branch is shared)

## Conflict Resolution During Rebase

When conflicts occur during rebase:

1. **Fix conflicts manually** in the marked files:

   ```
   <<<<<<< HEAD
   your changes
   =======
   incoming changes
   >>>>>>> commit-hash
   ```

2. **Stage the resolved files:**

   ```bash
   git add path/to/resolved/file
   ```

3. **Continue the rebase:**

   ```bash
   git rebase --continue
   ```

4. **Or abort if needed:**
   ```bash
   git rebase --abort
   ```

## Detailed Walkthrough Scenarios

### Scenario 1: The "Oops, I Diverged" Reset

**Problem:** Your local main is 3 commits ahead and 127 commits behind origin/main (like your original issue)

**What happened:** You made commits directly to main instead of a feature branch, and now your local main has diverged from the remote.

#### Command Line Solution:

```bash
# Step 1: See the problem
git status
# Output: "Your branch and 'origin/main' have diverged"

# Step 2: Fetch latest remote state
git fetch origin
# What this does: Downloads all remote changes without merging them

# Step 3: Nuclear reset to match remote exactly
git reset --hard origin/main
# What this does: Moves your HEAD to match origin/main exactly, discarding all local commits and changes
# Output: "HEAD is now at f16f033 progress"
```

#### VS Code/Cursor Solution:

1. **Status bar shows:** `main ↑3 ↓127` (3 ahead, 127 behind)
2. **Command Palette** (`Ctrl+Shift+P`) → "Git: Fetch"
3. **Source Control panel** shows "Your branch has diverged"
4. **Command Palette** → "Git: Reset" → Select "Hard" → Choose "origin/main"
5. **Confirm destructive action** in dialog
6. **Status bar now shows:** `main` (clean, no divergence)

**What this resolves:** Completely aligns your local branch with remote, losing any local commits you made by mistake.

---

### Scenario 2: Feature Branch Merge Conflict Hell

**Problem:** You've been working on `feature/user-auth` for 2 weeks. Main has moved forward with database changes, and now you have conflicts in migrations, models, and routes.

**The Setup:**

- **Your branch:** Added user authentication with Devise
- **Main branch:** Added admin panel with different user model changes
- **Conflict files:** `db/migrate/xxx_devise_create_users.rb`, `app/models/user.rb`, `config/routes.rb`

#### Command Line Walkthrough:

```bash
# Step 1: Get your bearings
git checkout feature/user-auth
git fetch origin

# Step 2: See what you're dealing with
git log --oneline main..HEAD
# Shows: Your 8 commits since branching from main

git log --oneline HEAD..origin/main
# Shows: 23 commits that happened in main while you worked

# Step 3: Start the rebase
git rebase origin/main
```

**First conflict appears in migration file:**

```ruby
# db/migrate/20250401120000_devise_create_users.rb
class DeviseCreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
<<<<<<< HEAD
      t.string :email,              null: false, default: ""
      t.string :first_name
      t.string :last_name
=======
      t.string :email,              null: false, default: ""
      t.string :username,           null: false
      t.boolean :admin,             default: false
>>>>>>> origin/main
      t.timestamps null: false
    end
  end
end
```

**Resolution steps:**

```bash
# Step 4: Manually resolve - combine both approaches
# Edit file to include all fields:
```

```ruby
create_table :users do |t|
  t.string :email,              null: false, default: ""
  t.string :username,           null: false
  t.string :first_name
  t.string :last_name
  t.boolean :admin,             default: false
  t.timestamps null: false
end
```

```bash
# Step 5: Stage the resolved file
git add db/migrate/20250401120000_devise_create_users.rb

# Step 6: Continue rebase
git rebase --continue
```

**Second conflict in User model:**

```ruby
# app/models/user.rb
class User < ApplicationRecord
<<<<<<< HEAD
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  validates :first_name, presence: true
  validates :last_name, presence: true
=======
  validates :username, presence: true, uniqueness: true

  scope :admins, -> { where(admin: true) }
>>>>>>> origin/main
end
```

**Resolution:**

```ruby
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  validates :username, presence: true, uniqueness: true
  validates :first_name, presence: true
  validates :last_name, presence: true

  scope :admins, -> { where(admin: true) }
end
```

```bash
git add app/models/user.rb
git rebase --continue
```

#### VS Code/Cursor Walkthrough:

1. **Start rebase:** Command Palette → "Git: Rebase Branch" → Select "origin/main"
2. **VS Code shows:** "Rebase in progress (1/8)" in status bar
3. **Source Control panel:** Shows `db/migrate/xxx_devise_create_users.rb` with "C" conflict icon
4. **Click the conflicted file** - opens in editor with colored sections:
   - **Blue (Current):** Your Devise fields
   - **Green (Incoming):** Their admin fields
5. **Click "Accept Both Changes"** then manually organize the fields
6. **Stage resolved file:** Click "+" next to file in Source Control
7. **Continue:** Command Palette → "Git: Continue Rebase"
8. **Repeat for User model conflict**
9. **Final result:** Status bar shows "feature/user-auth" (clean)

**What this resolves:** Your feature branch now has all main's changes incorporated, with your authentication feature cleanly applied on top.

---

### Scenario 3: The "Abandon Ship" Merge Abort

**Problem:** You started merging a complex feature branch but realize the conflicts are too gnarly and you need to rethink your approach.

**The Setup:**

```bash
git merge feature/complex-refactor
# Output:
# Auto-merging app/controllers/application_controller.rb
# CONFLICT (content): Merge conflict in app/controllers/application_controller.rb
# Auto-merging app/models/card.rb
# CONFLICT (content): Merge conflict in app/models/card.rb
# Auto-merging config/routes.rb
# CONFLICT (content): Merge conflict in config/routes.rb
# Automatic merge failed; fix conflicts and then commit the result.
```

#### Command Line Solution:

```bash
# Step 1: Realize this is too much
git status
# Output: Shows 3 files in "both modified" state

# Step 2: Abandon the merge entirely
git merge --abort
# What this does: Returns you to exactly where you were before the merge attempt

# Step 3: Verify you're clean
git status
# Output: "On branch main, nothing to commit, working tree clean"
```

#### VS Code/Cursor Solution:

1. **Source Control shows:** "Merge in progress" with 3 conflicted files
2. **Command Palette:** "Git: Abort Merge"
3. **Confirmation dialog:** "Are you sure you want to abort the merge?"
4. **Click "Yes"**
5. **Status bar returns to:** `main` (clean state)

**What this resolves:** Cleanly backs out of a problematic merge, returning you to a safe state to try a different approach (like rebasing the feature branch first, or breaking it into smaller pieces).

---

### Scenario 4: The "Keep Main's Version" Conflict Resolution

**Problem:** During a merge, you realize that main's version of certain files is actually what you want to keep (maybe your feature changes were experimental).

#### Command Line Solution:

```bash
# During a merge conflict, choose main's version for specific files
git checkout --ours config/database.yml    # Keep main's version
git checkout --theirs app/models/user.rb   # Keep feature branch version

# Or choose main's version for everything:
git checkout --ours .
git add .
git commit -m "Resolved conflicts keeping main's configuration"
```

#### VS Code/Cursor Solution:

1. **In conflict editor:** Click "Accept Incoming Change" for each conflict you want from main
2. **Or use Command Palette:** "Git: Accept All Incoming" to take all of main's changes
3. **Stage resolved files:** Click "+" or use "Stage All Changes"
4. **Commit:** Write message and press `Ctrl+Enter`

**What this resolves:** Quickly resolves conflicts when you know you want to favor one side, common when merging experimental branches or when configuration files have diverged.

---

## Git Muscle Memory Exercises

### Exercise 1: The Daily Workflow Drill

**Goal:** Make the basic branch workflow automatic

**Setup once:** Create a practice repo

```bash
mkdir git-practice && cd git-practice
git init
echo "# Practice Repo" > README.md
git add README.md && git commit -m "Initial commit"
git remote add origin https://github.com/yourusername/git-practice.git
```

**Daily 5-minute drill:** (Do this until it's muscle memory)

```bash
# 1. Status check (always start here)
git status

# 2. Create and switch to feature branch
git checkout -b feature/practice-$(date +%s)

# 3. Make a quick change
echo "Practice change $(date)" >> practice.txt

# 4. Stage and commit
git add .
git commit -m "Practice: add timestamp"

# 5. Switch back to main
git checkout main

# 6. Clean up the branch
git branch -d feature/practice-*
```

**Time goal:** Under 30 seconds without thinking

### Exercise 2: Conflict Resolution Bootcamp

**Setup:** Create a practice conflict scenario

```bash
# Terminal 1 (simulate teammate)
echo "original line" > conflict.txt
git add conflict.txt && git commit -m "Original"
git checkout -b teammate-branch
echo "teammate change" > conflict.txt
git add conflict.txt && git commit -m "Teammate change"

# Terminal 2 (simulate you)
git checkout main
echo "your change" > conflict.txt
git add conflict.txt && git commit -m "Your change"
git merge teammate-branch  # Creates conflict!
```

**Drill:** Resolve this conflict 10 times using different methods:

1. `git checkout --ours conflict.txt` (2 times)
2. `git checkout --theirs conflict.txt` (2 times)
3. Manual resolution combining both (3 times)
4. VS Code conflict editor (3 times)

**Time goal:** Recognize conflict type and resolve in under 60 seconds

### Exercise 3: The "Oh Shit" Recovery Drills

**Drill A:** Accidental commit to main

```bash
# Simulate the mistake
git checkout main
echo "oops" >> README.md
git add README.md && git commit -m "Accidental commit"

# Recovery drill - time yourself:
git reset --soft HEAD~1  # Undo commit, keep changes
git checkout -b proper-feature-branch
git commit -m "Proper feature commit"
```

**Drill B:** Diverged branches (your original problem)

```bash
# Simulate divergence
git checkout main
git reset --hard HEAD~3  # Go back 3 commits
echo "local work" >> local.txt
git add . && git commit -m "Local work"

# Recovery drill:
git fetch origin
git reset --hard origin/main
```

**Time goal:** Identify the problem and fix it in under 2 minutes

### Exercise 4: Rebase Muscle Memory

**Setup:** Create a branch with multiple commits

```bash
git checkout -b rebase-practice
for i in {1..3}; do
  echo "Change $i" >> file$i.txt
  git add file$i.txt
  git commit -m "Add file $i"
done
```

**Drill:** Practice interactive rebase

```bash
git rebase -i HEAD~3
# Practice: squash commits, reorder, edit messages
```

**VS Code equivalent drill:**

1. GitLens extension → "Rebase"
2. Practice squashing commits visually
3. Practice changing commit messages

### Exercise 5: Stash Gymnastics

**Scenario:** You're working on something but need to quickly switch contexts

**Drill sequence:**

```bash
# 1. Start work
echo "incomplete work" >> temp.txt

# 2. Emergency context switch
git stash push -m "WIP: fixing bug"
git checkout other-branch

# 3. Do emergency work
echo "hotfix" >> hotfix.txt
git add . && git commit -m "Emergency fix"

# 4. Return to original work
git checkout original-branch
git stash pop

# 5. Continue original work
echo "more work" >> temp.txt
git add . && git commit -m "Complete feature"
```

**Time goal:** Complete stash-work-unstash cycle in under 90 seconds

### Exercise 6: VS Code Speed Drills

**Goal:** Make GUI Git operations fast

**Daily VS Code drill:**

1. `Ctrl+Shift+G` (open Source Control) - 5 times
2. `Ctrl+Shift+P` → "Git: " → practice autocomplete for common commands
3. Stage files with `+` button - 10 times
4. Commit with `Ctrl+Enter` - 10 times
5. Branch switching via status bar - 5 times

### Exercise 7: Command Recall Challenge

**Test yourself:** Set a timer for 2 minutes and write out these commands from memory:

**Basic operations:**

- Check status
- Create and switch to new branch
- Stage all changes
- Commit with message
- Push to remote
- Pull latest changes

**Conflict resolution:**

- Abort a merge
- Reset hard to remote
- Accept "ours" for a file
- Continue a rebase

**Advanced:**

- Interactive rebase last 3 commits
- Stash with message
- Cherry-pick a commit
- Force push (safely)

**Scoring:**

- 15+ correct: Git ninja 🥷
- 10-14 correct: Git warrior ⚔️
- 5-9 correct: Git apprentice 🎓
- <5 correct: More practice needed! 💪

### Weekly Challenge: Real Project Simulation

**Friday exercise:** Create a realistic branching scenario:

1. **Main branch:** Start with a Rails app structure
2. **Feature branch 1:** Add authentication
3. **Feature branch 2:** Add admin panel
4. **Hotfix branch:** Fix critical bug
5. **Practice:** Merge conflicts between all branches

**Goal:** Complete realistic multi-branch workflow with conflicts in under 15 minutes

### Muscle Memory Mantras

**Before any Git operation:**

1. "Status first" - `git status`
2. "Fetch before rebase" - `git fetch origin`
3. "Branch before work" - `git checkout -b feature/name`

**During conflicts:**

1. "Understand the conflict" - Read the markers
2. "Test the resolution" - Make sure it works
3. "Stage and continue" - `git add` then `git rebase --continue`

**After any major operation:**

1. "Verify the result" - `git log --oneline`
2. "Check working directory" - `git status`
3. "Test the application" - Run your tests

## Quick Reference Table

| Goal               | Command                                                          |
| ------------------ | ---------------------------------------------------------------- |
| Stay clean, linear | `git rebase origin/main`                                         |
| Keep full history  | `git merge origin/main`                                          |
| Fix conflicts      | Edit files → `git add` → `git rebase --continue` or `git commit` |
| Bail out           | `git rebase --abort` or `git merge --abort`                      |

## Using VS Code & Cursor for Git Operations

Both VS Code and Cursor have excellent built-in Git support and use nearly identical interfaces:

### Force Reset to Remote (GUI Method)

1. **Open Source Control panel** (`Ctrl+Shift+G` / `Cmd+Shift+G`)
2. **Click the "..." menu** in Source Control
3. **Select "Pull, Push" → "Fetch"** to get latest remote refs
4. **Click branch name** in status bar → **"Switch to Another Branch"**
5. **Select your branch** and choose **"Reset Hard"** option
6. **Confirm** the destructive action

**Alternative:** Use the **Command Palette** (`Ctrl+Shift+P` / `Cmd+Shift+P`):

- Type "Git: Reset" → Select "Git: Reset (Hard)" → Choose "origin/main"

### Handling Merge Conflicts in Editor

When conflicts occur, VS Code/Cursor automatically:

1. **Highlights conflicted files** in Source Control with a "C" icon
2. **Opens conflict editor** with color-coded sections:

   - **Green:** Incoming changes (from main)
   - **Blue:** Current changes (your branch)
   - **Gray:** Common ancestor

3. **Conflict resolution buttons** appear above each conflict:
   - **"Accept Current Change"** (keep yours)
   - **"Accept Incoming Change"** (keep theirs)
   - **"Accept Both Changes"** (merge both)
   - **"Compare Changes"** (side-by-side view)

### Rails Controller Conflict Example in VS Code

When you hit the `cards_controller.rb` conflict, you'll see:

```ruby
def show
<<<<<<< HEAD (Current Change)
  @card = Card.find(params[:id])
  render layout: 'card'
=======
  Rails.logger.info("Showing card #{params[:id]}")    # Incoming Change
  @card = Card.find(params[:id])
>>>>>>> origin/main
end
```

**Click "Accept Both Changes"** then manually arrange:

```ruby
def show
  Rails.logger.info("Showing card #{params[:id]}")
  @card = Card.find(params[:id])
  render layout: 'card'
end
```

### Rebase in VS Code/Cursor

1. **Command Palette** → "Git: Rebase Branch"
2. **Select target branch** (e.g., "origin/main")
3. **Resolve conflicts** using the visual editor
4. **Stage resolved files** (click "+" next to file in Source Control)
5. **Continue rebase**: Command Palette → "Git: Continue Rebase"

### Merge vs Rebase GUI Indicators

**During Rebase:**

- Status bar shows: `Rebasing (1/3)` (commit 1 of 3)
- Source Control shows "Rebase in progress"

**During Merge:**

- Status bar shows: `Merging`
- Source Control shows "Merge in progress"

### Useful VS Code/Cursor Extensions

- **GitLens**: Enhanced Git capabilities, blame annotations, history
- **Git Graph**: Visual commit history and branch management
- **Git History**: File and line history viewer

### VS Code Git Commands via Command Palette

| Action             | Command                                  |
| ------------------ | ---------------------------------------- |
| Force reset        | "Git: Reset" → "Hard"                    |
| Abort merge/rebase | "Git: Abort Merge" / "Git: Abort Rebase" |
| Fetch              | "Git: Fetch"                             |
| Pull with rebase   | "Git: Pull (Rebase)"                     |
| View conflicts     | "Git: Open Changes"                      |

### Pro Tips for GUI Git

- **Use integrated terminal** (` Ctrl+``  ` ) for complex commands
- **Stage partial changes** by selecting lines in diff view
- **Commit with Ctrl+Enter** after writing message
- **View file history** with right-click → "Git: View File History"
- **Compare branches** via Git Graph extension
- **Stash changes** before switching branches: Command Palette → "Git: Stash"

## Pro Tips

- For ongoing development with fast-moving main: `git pull --rebase origin main`
- Always `git fetch origin` before rebasing or merging
- Use `git status` frequently to understand your current state
- When in doubt, create a backup branch: `git branch backup-feature-name`
- **VS Code/Cursor users**: Keep Source Control panel open while resolving conflicts
- **Use the visual diff**: Much easier than command line for complex conflicts
