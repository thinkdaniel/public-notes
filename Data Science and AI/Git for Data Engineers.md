# Git for Data Engineers: A Hands-On Tutorial

**Total Duration:** ~2 hours

**Prerequisites:**

- VS Code installed
- Git installed on your computer

---

## Section 1: Getting Started with Git (15 minutes)

### 1.1 Introduction

Git is a version control system that helps you track changes in your files. Think of it as a "save game" feature for your code - you can always go back to previous versions if something goes wrong.

**Why Git for Data Engineers?**

- Track changes in SQL queries, Python scripts, and notebooks
- Experiment with new approaches without losing your working code
- Collaborate with team members
- Keep a history of your data pipeline development

### 1.2 Initializing a Repository

**What we'll do:** Turn a regular folder into a Git-tracked folder

**Steps:**

1. **Create a new folder** called `git-demo` (use File Explorer/Finder or VS Code)

2. **Open the folder in VS Code**
   - File → Open Folder → Select `git-demo`

3. **Open the Terminal in VS Code**
   - Terminal → New Terminal (or press `` Ctrl+` ``)

4. **Check if this is a Git repository:**

   ```bash
   git status
   ```

   You'll see an error message: `fatal: not a git repository`
   - This confirms the folder is NOT yet tracked by Git
   - It's just a regular folder right now

5. **Initialize Git:**
   ```bash
   git init
   ```

**What just happened?**

- Git created a hidden `.git` folder that stores all your version history
- Your folder is now a **Git repository**
- Git is ready to track changes

6. **Check the status again:**

   ```bash
   git status
   ```

   Now you should see: "On branch main" and "No commits yet"
   - The error is gone! Git is now tracking this folder

**Exercise:** Follow the steps above to create and initialize your Git repository.

---

## Section 2: Understanding .gitignore (10 minutes)

### 2.1 Why .gitignore?

Some files shouldn't be tracked by Git:

- **Large data files** (CSV, Parquet files) - too big for Git
- **Passwords and API keys** - security risk!
- **Temporary files** - not important to track

### 2.2 Creating .gitignore

**Steps:**

1. **In VS Code:** Click "New File" button or right-click in Explorer
2. **Name it:** `.gitignore` (don't forget the dot at the beginning!)
3. **Add these patterns:**

```gitignore
# Data files - too large for Git
*.csv
*.parquet
*.json
data/

# Python temporary files
__pycache__/
*.pyc

# Jupyter Notebooks checkpoints
.ipynb_checkpoints/

# Passwords and keys - NEVER commit these!
.env
credentials.json
*.key

# Your IDE settings
.vscode/
.DS_Store
```

4. **Save the file** (Ctrl+S or Cmd+S)

**What this does:**

- Git will now ignore any file matching these patterns
- You won't accidentally commit large data files or passwords

**Exercise:** Create a `.gitignore` file in your repository with the patterns above.

---

## Section 3: The Git Workflow - From Changes to History (20 minutes)

### 3.1 Understanding the Three Stages

When you work with Git, your files go through three stages:

```
1. Working Directory → 2. Staging Area → 3. Repository (Committed)
   (You edit here)      (Prepare to save)    (Permanently saved)
```

**Think of it like:**

1. **Working Directory** = Your shopping cart (adding/removing items)
2. **Staging Area** = Checkout line (deciding what to buy)
3. **Repository** = Receipt (permanent record of purchase)

### 3.2 Hands-On: Your First Commit

**Step 1: Create a file in VS Code**

1. Click "New File" in VS Code
2. Name it: `note1.txt`
3. Type inside: `This is my first note about data engineering`
4. Save the file (Ctrl+S or Cmd+S)

**Step 2: Check Git status**

In the Terminal:

```bash
git status
```

You'll see `note1.txt` in red under "Untracked files"

- **Red** = Git sees the file but isn't tracking it yet

**Step 3: Add to Staging Area**

```bash
git add note1.txt
```

Check status again:

```bash
git status
```

Now `note1.txt` is in green under "Changes to be committed"

- **Green** = Ready to be saved permanently

**Step 4: Commit to Repository**

```bash
git commit -m "Add first note about data engineering"
```

Check status:

```bash
git status
```

Should say: "nothing to commit, working tree clean"

- Your change is now permanently saved in Git history!

### 3.3 Working with Multiple Files

Let's practice the workflow with more files:

1. **Create 3 new text files in VS Code:**
   - `query1.txt` → Type: `SELECT * FROM customers;`
   - `query2.txt` → Type: `SELECT * FROM orders;`
   - `README.txt` → Type: `My SQL Query Collection`

2. **Check what Git sees:**

   ```bash
   git status
   ```

   All three files should be red (untracked)

3. **Add all files at once:**

   ```bash
   git add .
   ```

   The dot (`.`) means "add everything in this folder"

4. **Commit them:**
   ```bash
   git commit -m "Add SQL queries and README"
   ```

### 3.4 Viewing Your History

See all your commits:

```bash
git log
```

Too much information? Try this simpler view:

```bash
git log --oneline
```

Each line shows:

- A unique ID (like `a3f5c2b`)
- Your commit message

**Exercise:** Create 3-4 text files and commit them using the workflow above.

---

## Section 4: Understanding Commits (10 minutes)

### 4.1 What is a Commit?

A **commit** is like taking a snapshot of your project at a specific moment. Each commit remembers:

- What files changed
- Who made the changes
- When it happened
- Why (your commit message)

### 4.2 Writing Good Commit Messages

**Bad commit messages:** ❌

```bash
git commit -m "updates"
git commit -m "fix"
git commit -m "asdf"
```

**Good commit messages:** ✅

```bash
git commit -m "Add customer data validation query"
git commit -m "Fix typo in SELECT statement"
git commit -m "Update README with query descriptions"
```

**Tips for good messages:**

- Be specific about what changed
- Use present tense: "Add" not "Added"
- Keep it short but descriptive
- Imagine you're reading this 6 months from now

**Exercise:** Look at your commit history (`git log --oneline`). Are your messages clear?

---

## Section 5: Undoing Changes with Git Reset (15 minutes)

### 5.1 Why We Need to Undo

Common situations:

- "Oops, I committed the wrong file!"
- "I want to change my commit message"
- "I need to start over on this change"

### 5.2 Three Ways to Reset

#### Option 1: Soft Reset (Keep Your Changes Ready)

Goes back one commit but keeps your files ready to commit again

```bash
git reset --soft HEAD~1
```

**Where is the file after reset?** ✅ **Staging Area** (ready to commit again)

- Your changes are still there
- They're still in green when you run `git status`
- Just need to run `git commit` again

**When to use:** You want to edit the commit message or add more files

#### Option 2: Mixed Reset (Keep Your Changes, But Unstage)

Goes back one commit, keeps your changed files but unstages them

```bash
git reset HEAD~1
```

**Where is the file after reset?** ✅ **Working Directory** (not staged)

- Your changes are still there
- They're in red when you run `git status`
- You need to run `git add` and then `git commit` again

**When to use:** You want to reorganize which files go into which commit

**💡 This is the most commonly used reset!** It's the default (you don't need `--mixed` flag) because it lets you:
- Review and edit your changes before recommitting
- Split one commit into multiple smaller commits
- Add more changes before committing again

#### Option 3: Hard Reset (⚠️ Delete Everything)

Goes back one commit and **deletes all your changes**

```bash
git reset --hard HEAD~1
```

**Where is the file after reset?** ❌ **Gone!** (deleted completely)

- Your changes are DELETED
- The file goes back to its previous committed state
- You CANNOT get your changes back

**When to use:** You want to completely abandon your recent work
**WARNING:** You'll lose your changes permanently!

### 5.3 Practice Undoing

Let's practice with mixed reset (the most common reset type):

1. **Create and commit a file:**
   - Create `work.txt` in VS Code
   - Type: `Draft of my analysis`
   - Save it
   - Add and commit:
     ```bash
     git add work.txt
     git commit -m "Add analysis draft"
     ```

2. **Undo with mixed reset:**
   ```bash
   git reset HEAD~1
   ```
   
   **Why mixed reset?** It gives you the most flexibility - your changes are safe but unstaged, so you can review, edit, or reorganize them before committing again.

3. **Check status:**

   ```bash
   git status
   ```

   Your file is still there but unstaged (red)!
   - **The file is in the Working Directory** - you can edit it before staging again

4. **Edit the file in VS Code:**
   - Change text to: `Complete analysis with findings`
   - Save it

5. **Add and commit again with better message:**
   ```bash
   git add work.txt
   git commit -m "Add complete analysis with findings"
   ```

**Exercise:** Practice each type of reset to understand the differences.

---

## Section 6: Git Branching (20 minutes)

### 6.1 What are Branches?

Imagine you're writing a book:

- **Main branch** = Your published chapters (working version)
- **Feature branch** = A draft where you experiment with new ideas

Branches let you work on new features without breaking your working code.

**Common use for data engineers:**

- Main branch = Working data pipeline
- Feature branch = Testing a new transformation logic

### 6.2 Creating Your First Branch

**See your current branch:**

```bash
git branch
```

You should see: `* main` (the asterisk shows where you are)

**Create a new branch:**

```bash
git branch feature-new-queries
```

**Switch to the new branch:**

```bash
git switch feature-new-queries
```

Or do both in one command:

```bash
git switch -c feature-new-queries
```

**Verify you switched:**

```bash
git branch
```

Now the asterisk should be on `feature-new-queries`

### 6.3 Working on Your Branch

Let's create something on our feature branch:

1. **Create a new file in VS Code:**
   - Create `advanced_query.txt`
   - Type: `SELECT customers.name, orders.total FROM customers JOIN orders;`
   - Save it

2. **Commit on the feature branch:**

   ```bash
   git add advanced_query.txt
   git commit -m "Add advanced join query"
   ```

3. **Switch back to main:**

   ```bash
   git switch main
   ```

4. **Look at your files in VS Code:**
   - `advanced_query.txt` disappeared! 😱
   - Don't worry, it's safe on the other branch

5. **Switch back to feature branch:**

   ```bash
   git switch feature-new-queries
   ```

   - `advanced_query.txt` is back! 😊

**What's happening?**

- Each branch has its own version of your files
- Git swaps the files when you switch branches
- Changes on one branch don't affect the other

### 6.4 Visualizing Branches

See your branches with history:

```bash
git log --oneline --graph --all
```

This shows a visual tree of your commits and branches!

**Exercise:** Create a feature branch, add 2-3 text files with SQL queries or notes, commit them, then switch between main and feature branches to see the files appear and disappear.

---

## Section 7: Merging Branches (20 minutes)

### 7.1 Why Merge?

Once your feature is complete and tested, you want to bring it back into the main branch.

**Merging = Combining the changes from two branches**

### 7.2 Simple Merge (Fast-Forward)

This happens when main hasn't changed since you created the feature branch.

**Let's do it:**

1. **Make sure you're on the feature branch:**

   ```bash
   git switch feature-new-queries
   ```

2. **Add another file:**
   - Create `query_final.txt` in VS Code
   - Type: `SELECT * FROM products WHERE price > 100;`
   - Save and commit:
     ```bash
     git add query_final.txt
     git commit -m "Add product price query"
     ```

3. **Switch to main:**

   ```bash
   git switch main
   ```

4. **Merge the feature branch into main:**

   ```bash
   git merge feature-new-queries
   ```

5. **Check your files:**
   - All the files from feature branch are now in main!
   - Check the log:
     ```bash
     git log --oneline
     ```

**What happened?**

- Git "fast-forwarded" main to include all feature branch commits
- No conflicts because main hadn't changed

### 7.3 Merge with Merge Commit

This happens when both branches have new commits - they've diverged.

**Let's set this up:**

1. **Create and switch to a new branch:**

   ```bash
   git switch -c feature-analysis
   ```

2. **On feature branch, create a file:**
   - Create `analysis.txt`
   - Type: `Data analysis notes`
   - Commit:
     ```bash
     git add analysis.txt
     git commit -m "Add analysis notes"
     ```

3. **Switch back to main:**

   ```bash
   git switch main
   ```

4. **On main, create a different file:**
   - Create `main_note.txt`
   - Type: `Important note on main branch`
   - Commit:
     ```bash
     git add main_note.txt
     git commit -m "Add important note on main"
     ```

   **Now both branches have diverged!** Main has `main_note.txt` and feature-analysis has `analysis.txt`

5. **Merge the feature branch into main:**

   ```bash
   git merge feature-analysis
   ```

   **What happens next:**
   - A text editor will open asking you to write a merge commit message
   - It will have a default message like `Merge branch 'feature-analysis'`
   - You can keep the default or edit it
   - **In VS Code:** Just save the file (Ctrl+S or Cmd+S) and close the tab
   - **In a terminal editor (Vim):** Press `Esc`, type `:wq`, press Enter
   
   **On success, you'll see:**
   ```
   Merge made by the 'ort' strategy.
   ```
   
   **What is 'ort' strategy?** It's Git's default merge algorithm (since Git 2.34). "Ort" stands for "Ostensibly Recursive's Twin" - it's a faster, more efficient version of Git's recursive merge strategy. For most users, you don't need to worry about it - it just means Git successfully combined your branches automatically!

6. **Look at the log:**

   ```bash
   git log --oneline --graph
   ```

   You'll see a **merge commit** that joins the two branches!

**Exercise:** Practice both types of merges.

### 7.4 Cleaning Up Merged Branches

Once you've merged a feature branch back into main, the branch has served its purpose. It's good practice to delete it to keep your branch list clean and manageable.

**Why delete merged branches?**

- Keeps `git branch` output clean and readable
- Prevents confusion about which branches are active
- The commits are already in main - nothing is lost!

**Check which branches you have:**

```bash
git branch
```

You might see something like:

```
  feature-new-queries
  feature-analysis
* main
```

**Delete a merged branch (safe):**

```bash
git branch -d feature-new-queries
```

- The `-d` flag means "delete"
- Git will **prevent you** from deleting if the branch isn't fully merged (safe!)
- You'll see: `Deleted branch feature-new-queries`

**What if Git refuses to delete?**

If you see an error like:

```
error: The branch 'feature-xyz' is not fully merged.
```

This means:

- The branch has commits that aren't in main yet
- Git is protecting you from losing work
- **Options:**
  - Merge the branch first, then delete it
  - Or use force delete (next section) if you're sure

**Force delete a branch (use carefully!):**

```bash
git branch -D feature-practice
```

- The `-D` flag (capital D) means "force delete"
- Use this when you want to delete a branch with unmerged work
- **Warning:** You'll lose any commits that aren't in main!
- Only use this if you're certain you don't need those changes

**Important reassurance:**

When you delete a branch after merging:

- All the commits are **still in main's history**
- Your work is **not lost**
- You're just removing the branch label/pointer
- The branch name is gone, but the code lives on in main

**Quick practice:**

1. **List your branches:**

   ```bash
   git branch
   ```

2. **Delete a branch you've already merged:**

   ```bash
   git branch -d feature-analysis
   ```

3. **Verify it's gone:**

   ```bash
   git branch
   ```

4. **Check that the commits are still in main:**
   ```bash
   git log --oneline
   ```
   You'll see all the commits from the deleted branch!

**When to keep a branch:**

- You're still working on it
- You plan to continue development later
- It's not ready to merge yet

**When to delete a branch:**

- It's been successfully merged
- You don't need to work on it anymore
- You want to start fresh with a new branch for the next feature

**Exercise:** Practice the complete workflow: Create a branch → Make commits → Merge to main → Delete the branch → Verify commits are still in history.

---

## Section 8: Handling Merge Conflicts (25 minutes)

### 8.1 What is a Merge Conflict?

A conflict happens when:

- The **same lines** in the **same file** are changed differently in two branches
- Git doesn't know which version to keep

**Example:**

- Main branch changes line 1 to "Version A"
- Feature branch changes line 1 to "Version B"
- Git says: "You decide which one to keep!"

### 8.2 Creating a Conflict (On Purpose)

Let's intentionally create a conflict to learn how to fix it:

**Step 1: Create the initial file on main**

1. Make sure you're on main:

   ```bash
   git switch main
   ```

2. Create `config.txt` in VS Code:

   ```
   Database: PostgreSQL
   Version: 1.0
   ```

3. Save and commit:
   ```bash
   git add config.txt
   git commit -m "Add initial config"
   ```

**Step 2: Create a feature branch and change the file**

1. Create and switch to feature branch:

   ```bash
   git switch -c feature-update-config
   ```

2. Edit `config.txt` in VS Code - change it to:

   ```
   Database: PostgreSQL
   Version: 2.0
   Feature: Enhanced
   ```

3. Save and commit:
   ```bash
   git add config.txt
   git commit -m "Update config to version 2.0"
   ```

**Step 3: Go back to main and make a DIFFERENT change**

1. Switch to main:

   ```bash
   git switch main
   ```

2. Edit `config.txt` in VS Code - change it to:

   ```
   Database: PostgreSQL
   Version: 1.5
   Debug: Enabled
   ```

3. Save and commit:
   ```bash
   git add config.txt
   git commit -m "Update config to version 1.5"
   ```

**Step 4: Try to merge - CONFLICT!**

```bash
git merge feature-update-config
```

You'll see:

```
Auto-merging config.txt
CONFLICT (content): Merge conflict in config.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Don't panic! This is normal. 😊

### 8.3 Understanding the Conflict

**Look at `config.txt` in VS Code.** You'll see something like:

```
Database: PostgreSQL
<<<<<<< HEAD
Version: 1.5
Debug: Enabled
=======
Version: 2.0
Feature: Enhanced
>>>>>>> feature-update-config
```

**What do these markers mean?**

- `<<<<<<< HEAD` = Start of YOUR version (current branch - main)
- `=======` = Separator
- `>>>>>>> feature-update-config` = End of THEIR version (merging branch)

Git is asking: **"Which version do you want to keep?"**

### 8.4 Resolving the Conflict

**Option 1: Manually decide what to keep**

Edit `config.txt` in VS Code to what you want. For example:

```
Database: PostgreSQL
Version: 2.0
Debug: Enabled
Feature: Enhanced
```

**Important:** Delete all the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)

**Option 2: Use VS Code's conflict resolver**

VS Code shows buttons above the conflict:

- **Accept Current Change** - Keep main branch version
- **Accept Incoming Change** - Keep feature branch version
- **Accept Both Changes** - Keep both (one after another)
- **Compare Changes** - See side-by-side comparison

Click the button you want!

**After resolving:**

1. Save the file

2. Check status:

   ```bash
   git status
   ```

3. Add the resolved file:

   ```bash
   git add config.txt
   ```

4. Complete the merge:

   ```bash
   git commit -m "Merge feature-update-config and resolve conflicts"
   ```
   
   **Important:** This creates a **merge commit** regardless of how you resolved the conflict (manually or with VS Code buttons). The merge commit exists because the two branches diverged, not because of the conflict itself.

5. Check the log:
   ```bash
   git log --oneline --graph
   ```

### 8.5 If You Want to Cancel the Merge

If you decide "I don't want to merge anymore":

```bash
git merge --abort
```

This cancels the merge and goes back to before you tried to merge.

**Exercise:** Create your own conflict scenario with a different file. Practice resolving it using both manual editing and VS Code's buttons.

---

## Section 9: Keeping Your Feature Branch Updated (15 minutes)

### 9.1 The Problem

Imagine this scenario:

- You create a feature branch on Monday
- You work on your feature for a week
- Meanwhile, your teammates add 20 commits to main
- When you try to merge your feature back... BIG CONFLICTS! 😱

**Solution:** Regularly update your feature branch with main's changes

### 9.2 Method 1: Merge Main into Your Feature (Recommended for Beginners)

This is the safer approach:

1. **Switch to your feature branch:**

   ```bash
   git switch feature-update-config
   ```

2. **Merge main into your feature:**

   ```bash
   git merge main
   ```

3. **Resolve any conflicts** (like we learned in Section 8)

4. **Continue working on your feature**

**Why this works:**

- Your feature branch now has all the latest changes from main
- When you eventually merge feature back to main, there will be fewer conflicts
- You test that your feature works with the latest main code

**When to do this:**

- Daily, if main changes frequently
- Before you finish your feature
- Before creating a pull request (in team settings)

### 9.3 Method 2: Rebase (Brief Mention)

There's another way called "rebase" that makes a cleaner history:

```bash
git switch feature-branch
git rebase main
```

**What's different:**

- Rebase "replays" your commits on top of main
- Creates a linear history (looks cleaner)
- More advanced - can be confusing for beginners

**For now:** Stick with the merge method above. You can learn rebase later!

### 9.4 Practice Scenario

Let's simulate keeping a feature branch updated:

1. **Create a feature branch:**

   ```bash
   git switch main
   git switch -c feature-practice
   ```

2. **Add a file on feature:**
   - Create `feature_work.txt`
   - Type: `Working on my feature`
   - Commit it

3. **Switch to main and add something:**

   ```bash
   git switch main
   ```

   - Create `main_update.txt`
   - Type: `Important main update`
   - Commit it

4. **Update your feature with main's changes:**

   ```bash
   git switch feature-practice
   git merge main
   ```

5. **Check your files:**
   - You should now have both `feature_work.txt` AND `main_update.txt`!

**Exercise:** Practice this flow several times with different files.

---

## Section 10: Git and Jupyter Notebooks - Important Notes (10 minutes)

### 10.1 The Problem with Jupyter Notebooks

Jupyter notebooks (`.ipynb` files) are challenging in Git because:

1. **They're actually JSON files** - not simple text

   ```json
   {
     "cells": [...],
     "outputs": [...],  ← This causes problems!
     "metadata": {...}
   }
   ```

2. **Outputs are included** - graphs, tables, images
   - Makes files very large
   - Creates huge diffs even for small code changes
   - Hard to see what actually changed

3. **Merge conflicts are nearly impossible to resolve**
   - The JSON structure is complex
   - Conflict markers break the file format

### 10.2 The Simple Solution: Clear Outputs Before Committing

**Best practice for beginners:**

**Before committing a notebook:**

1. Open the notebook in Jupyter/VS Code
2. Go to: **Edit → Clear All Outputs** (or Cell → All Output → Clear)
3. Save the notebook
4. Then commit it with Git

**Why this helps:**

- Notebook file is much smaller
- Git diffs show only your code changes
- Easier to see what changed
- Merge conflicts are less likely

**Example workflow:**

```bash
# After working on your notebook:
# 1. Manually clear outputs in Jupyter (Edit → Clear All Outputs)
# 2. Save the notebook
# 3. Then commit:

git add analysis.ipynb
git commit -m "Add customer analysis notebook"
```

### 10.3 Additional Notes

**For production work:**

- Move reusable code from notebooks to `.py` files
- Use notebooks only for exploration and analysis
- Track `.py` files in Git instead

**Project structure tip:**

```
project/
├── notebooks/         ← Exploratory analysis
│   └── analysis.ipynb
├── src/              ← Production code (track this!)
│   ├── data_processing.py
│   └── etl.py
└── tests/
    └── test_processing.py
```

**Advanced tools** (just for awareness, not required now):

- `nbstripout` - Automatically clears outputs when committing
- `nbdime` - Better diff tool for notebooks
- `ReviewNB` - Web-based notebook diff viewer

**For now:** Just remember to **manually clear outputs before committing!**

**Exercise:** If you have a Jupyter notebook, practice clearing outputs and committing it.

---

## Section 11: (Optional) Pushing to GitHub (15 minutes)

### 11.1 Why Use GitHub?

So far, we've been working with **local Git** - everything is on your computer.

**GitHub** adds cloud storage and collaboration:

- ☁️ **Backup** - Your code is safe even if your computer breaks
- 👥 **Collaboration** - Share code with teammates
- 📱 **Access anywhere** - Work from any computer
- 🔍 **Portfolio** - Show your work to potential employers

**Think of it as:**

- **Local Git** = Saving your game on your console
- **GitHub** = Cloud save that syncs across devices

### 11.2 Creating a Repository on GitHub

**Prerequisites:** You need a GitHub account (free at github.com)

**Steps:**

1. **Go to GitHub.com** and log in

2. **Click the "+" icon** in the top right → "New repository"

3. **Fill in the details:**
   - Repository name: `git-demo`
   - Description: `Learning Git for data engineering`
   - Choose: **Public** or **Private**
   - **Do NOT** check "Initialize with README" (we already have commits locally)

4. **Click "Create repository"**

5. **You'll see a page with instructions** - we'll use VS Code to push our code!

### 11.3 Pushing to GitHub - VS Code Method (Easiest!)

**VS Code makes pushing to GitHub incredibly simple:**

**First time setup:**

1. **Sign in to GitHub in VS Code** (if not already)
   - Click the account icon in bottom left
   - "Sign in to sync settings" → Sign in with GitHub

2. **Open Source Control** (left sidebar, or Ctrl/Cmd+Shift+G)

3. **Click "Publish Branch"** button
   - VS Code will prompt you to select the repository name
   - Choose whether to make it public or private
   - That's it! Your code is now on GitHub! 🎉

**What VS Code does automatically:**
- Creates the remote connection to GitHub
- Pushes all your commits
- Handles authentication
- Sets up tracking for future pushes

**For subsequent pushes:**
- Make your changes and commit them (Source Control panel)
- Click the **"Sync Changes"** button (or the ↑ arrow with a number)
- Done!

**Advantages:**
- No need to remember commands
- Visual interface
- Automatic authentication
- Clear indicators of what will be pushed

### 11.4 Pushing to GitHub - Command Line Method (Alternative)

**If you prefer the command line, here's how:**

**In your VS Code terminal:**

1. **Add GitHub as a "remote"** (a remote is a cloud copy):

   ```bash
   git remote add origin https://github.com/YOUR-USERNAME/git-demo.git
   ```

   Replace `YOUR-USERNAME` with your actual GitHub username!

2. **Verify the remote was added:**

   ```bash
   git remote -v
   ```

   You should see:

   ```
   origin  https://github.com/YOUR-USERNAME/git-demo.git (fetch)
   origin  https://github.com/YOUR-USERNAME/git-demo.git (push)
   ```

**What is "origin"?**

- Just a nickname for your GitHub repository
- You could name it anything, but "origin" is the standard name

2. **Push your commits:**

**Push = Upload your commits to GitHub**

```bash
git push -u origin main
```

**What this does:**

- `push` - Upload commits
- `-u origin main` - Set GitHub's main branch as the default destination
- Next time, you can just type `git push`

**You might be asked to:**

- Enter your GitHub username
- Enter a password or Personal Access Token (GitHub no longer accepts passwords - you need to create a token in GitHub Settings → Developer settings → Personal access tokens)

**After pushing:** Refresh your GitHub repository page - your files are there! 🎉

### 11.5 The Daily Workflow with GitHub

Once connected, your workflow becomes:

**Start of the day:**

```bash
git pull
```

Downloads any changes from GitHub (useful when collaborating)

**During work:**

```bash
# Work on files
git add .
git commit -m "Your message"
```

**End of the day (or after each feature):**

```bash
git push
```

Uploads your commits to GitHub

**Or in VS Code:** Just click the "Sync Changes" button!

### 11.6 Cloning a Repository

**Clone = Download a repository from GitHub to your computer**

If you want to work on another computer:

```bash
git clone https://github.com/YOUR-USERNAME/git-demo.git
cd git-demo
```

Now you have a complete copy with all history!

**Or in VS Code:** Command Palette (Ctrl/Cmd+Shift+P) → "Git: Clone" → paste the URL

### 11.7 Quick Reference

```bash
# Setup (one time)
git remote add origin https://github.com/USER/REPO.git

# Check remotes
git remote -v

# Upload commits to GitHub
git push                    # After first push
git push -u origin main    # First time only

# Download updates from GitHub
git pull

# Download a repository
git clone https://github.com/USER/REPO.git
```

**VS Code shortcuts:**
- Publish Branch: First time pushing a new branch
- Sync Changes: Push and pull in one action
- Command Palette → "Git: Clone": Clone a repository

### 11.8 Common Issues

**Problem:** `git push` asks for password repeatedly

**Solutions:**

- Use SSH keys (more advanced)
- Use GitHub Desktop app (easier for beginners)
- Use VS Code's built-in Git features (easiest!)

**Problem:** Can't push because someone else made changes

**Solution:**

```bash
git pull              # Download their changes
# Fix any conflicts if needed
git push              # Now push your changes
```

**Problem:** Forgot to pull before making changes

**Solution:**

```bash
git pull              # Might create merge conflicts
# Resolve conflicts
git commit
git push
```

**Note:** Most of these issues are avoided when using VS Code's UI, as it handles conflicts and syncing visually!

### 11.9 Next Steps with GitHub

After mastering basic push/pull:

- **Branches on GitHub** - Create branches for features
- **Pull Requests** - Request code review before merging
- **GitHub Actions** - Automate testing and deployment
- **Issues** - Track bugs and features
- **GitHub Pages** - Host documentation websites

**For Data Engineers specifically:**

- Share data pipeline code with team
- Collaborate on SQL queries and transformations
- Version control DAG files (Airflow, Prefect)
- Document data models and schemas

---

## Summary and Best Practices (5 minutes)

### 11.1 Quick Command Reference

```bash
# Setup
git init                        # Start tracking a folder
git status                      # Check current state

# Basic workflow
git add <file>                  # Stage a file
git add .                       # Stage all files
git commit -m "message"         # Save a snapshot
git log --oneline               # View history

# Undoing
git restore <file>              # Discard changes in file
git restore --staged <file>     # Unstage a file
git reset --soft HEAD~1         # Undo commit, keep changes staged
git reset HEAD~1                # Undo commit, keep changes unstaged
git reset --hard HEAD~1         # Undo commit, delete changes ⚠️

# Branching
git branch                      # List branches
git branch <name>               # Create branch
git switch <name>               # Switch to branch
git switch -c <name>            # Create and switch
git merge <branch>              # Merge branch into current
git merge --abort               # Cancel a merge

# Keeping feature updated
git switch feature-branch
git merge main                  # Bring main changes into feature
```

### 11.2 Best Practices for Data Engineers

1. **Commit often** - Small commits are easier to understand and undo
2. **Write clear messages** - Your future self will thank you
3. **Use .gitignore** - Don't commit data files or credentials
4. **Clear notebook outputs** - Before committing any `.ipynb` file
5. **Use branches** - Even for small experiments
6. **Keep main stable** - Only merge working, tested code
7. **Update feature branches regularly** - Merge main into feature daily
8. **Review before committing** - Check `git status` and files changed

### 11.3 Common Mistakes to Avoid

❌ Committing large CSV or data files
✅ Use `.gitignore` and store data elsewhere

❌ Committing passwords or API keys
✅ Put them in `.gitignore` and use environment variables

❌ Vague commit messages like "updates" or "fix"
✅ Be specific: "Fix null handling in ETL script"

❌ Working directly on main branch
✅ Create a feature branch for each task

❌ Forgetting to clear notebook outputs
✅ Always clear before committing

### 11.4 What's Next?

After mastering local Git, you can learn:

- **GitHub/GitLab** - Sharing code with others
- **Pull Requests** - Team code review
- **Cloning** - Getting code from others
- **Remote repositories** - Backup and collaboration
- **Git for teams** - Workflows for data engineering teams

---
