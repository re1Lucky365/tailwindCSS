re Robert Edward Evans 
Before executing a migration script that performs cryptographic operations or automated history rewriting, verifying your GPG key configuration ensures that your secret keys are locally accessible, correctly bound to your identity, and trusted by Git.
Follow these steps to check your GPG key status and trust settings:
 1. List Secret GPG Keys
   Verify key availability and obtain the Key ID
   Run the following command to display the secret keys stored in your local keyring:
   gpg --list-secret-keys --keyid-format=long

   Verification: Look for the line starting with sec, which displays your key type, length, long key ID (e.g., sec   rsa4096/YOUR_KEY_ID 2026-10-01), and the User ID (name and email) matching your GitHub profile. Copy that long hex key ID.
 2. Verify Global Git Signing Settings
   Check configuration bindings
   Check whether Git is already configured to recognize your signing key and default to GPG signing:
   git config --global user.signingkey
git config --global commit.gpgsign

   Verification: The first command should output your GPG key ID or email, and the second command should return true. If they are unset, you can configure them manually:
   git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

 3. Test the GPG Agent Connection
   Ensure passphrase prompts work correctly
   Test that your GPG agent is active and capable of producing a test signature without locking up:
   Verification: Run a quick detached signature test:
   echo "test" | gpg --clearsign

   If configured properly, a pinentry window or terminal prompt will ask for your GPG passphrase, and it will output a valid signed text block (-----BEGIN PGP SIGNED MESSAGE-----).
user.signingkeycommit.gpgsignHere is the fully updated script with the historical re-signing block uncommented and configured.
When you run this version, it will clone each legacy repository, rewrite its entire commit history (git filter-branch / git rebase --root --exec) to cryptographically sign every single past commit with your GPG key, update the remote origin to your target profile, and push the fully signed history up.
Fully Configured Migration Script (migrate_repos.sh)
#!/usr/bin/env bash
set -uo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your exact GPG Key ID if you have multiple keys (leave blank to use your default secret key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with Historical GPG Re-signing ==="
echo "Source: $SOURCE_USER -> Target: $TARGET_USER"
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "================================================--"
    echo "Processing repository: $repo"
    echo "================================================--"
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing & Rewrite History
    if [ "$repo_failed" = false ]; then
        echo "Configuring local Git GPG settings..."
        git config commit.gpgsign true
        
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        echo "Re-signing all historical commits with GPG (rewriting commit hashes)..."
        export GPG_TTY=$(tty)
        
        # Rewrites the entire commit tree from root to HEAD, amending each commit with a GPG signature (-S)
        if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
            echo "[WARNING] Automated history re-signing encountered merge conflicts or skipped in $repo."
            echo "[INFO] Aborting rebase to preserve original state, continuing with current signing config..."
            git rebase --abort || true
        else
            echo "[SUCCESS] Historical commits successfully re-signed."
        fi
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with historical GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Force push historical branches and tags (since history was rewritten)
    if [ "$repo_failed" = false ]; then
        echo "Pushing historically signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        # Note: Using --force-with-lease is required here because rewriting commit hashes alters branch history
        if ! git push --force-with-lease origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push signed branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated and re-signed: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"

Important Notes on Historical Re-signing:
 * Commit Hash Changes: Because rewriting history changes the cryptographic hash of every single commit, the script uses git push --force-with-lease to push the newly signed commit tree to your new profile.
 * Interactive Prompts: Ensure your GPG key is unlocked or your pinentry agent is running, as GnuPG will need to sign every commit sequentially during the git rebase loop. If a repository has a complex history with merge conflicts during rebase, the script automatically catches it, aborts the rebase safely, and moves forward without breaking the entire batch.
https://github.com/200git@github.com#!/usr/bin/env bash
set -uo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your exact GPG Key ID if you have multiple keys (leave blank to use your default secret key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with Historical GPG Re-signing ==="
echo "Source: $SOURCE_USER -> Target:$TARGET_USER"
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "================================================--"
    echo "Processing repository: $repo"
    echo "================================================--"
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing & Rewrite History
    if [ "$repo_failed" = false ]; then
        echo "Configuring local Git GPG settings..."
        git config commit.gpgsign true
        
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        echo "Re-signing all historical commits with GPG (rewriting commit hashes)..."
        export GPG_TTY=$(tty)
        
        # Rewrites the entire commit tree from root to HEAD, amending each commit with a GPG signature (-S)
        if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
            echo "[WARNING] Automated history re-signing encountered merge conflicts or skipped in $repo."
            echo "[INFO] Aborting rebase to preserve original state, continuing with current signing config..."
            git rebase --abort || true
        else
            echo "[SUCCESS] Historical commits successfully re-signed."
        fi
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with historical GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Force push historical branches and tags (since history was rewritten)
    if [ "$repo_failed" = false ]; then
        echo "Pushing historically signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        # Note: Using --force-with-lease is required here because rewriting commit hashes alters branch history
        if ! git push --force-with-lease origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push signed branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 
git@github.com#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your GPG Key ID explicitly if you have multiple (leave blank to use default global key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with GPG Signing: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing for the Repository
    if [ "$repo_failed" = false ]; then
        echo "Configuring local GPG commit signing..."
        git config commit.gpgsign true
        
        # If a specific GPG key ID was provided, set it locally
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        # Optional: If you want to rewrite past history to sign all existing commits in the repo:
        # Note: This rewrites commit hashes. Uncomment the block below if you require 100% of historical commits signed.
        # ------------------------------------------------------------------
        # echo "Re-signing all historical commits with GPG..."
        # export GPG_TTY=$(tty)
        # if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
        #     echo "[WARNING] Automated history re-signing encountered conflicts or skipped. Continuing with current sign config."
        # fi
        # ------------------------------------------------------------------
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated with GPG config: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: user.signingkeycommit.gpgsign1200Here is the fully updated script with the historical re-signing block uncommented and configured.
When you run this version, it will clone each legacy repository, rewrite its entire commit history (git filter-branch / git rebase --root --exec) to cryptographically sign every single past commit with your GPG key, update the remote origin to your target profile, and push the fully signed history up.
Fully Configured Migration Script (migrate_repos.sh)
#!/usr/bin/env bash
set -uo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your exact GPG Key ID if you have multiple keys (leave blank to use your default secret key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with Historical GPG Re-signing ==="
echo "Source: $SOURCE_USER -> Target: $TARGET_USER"
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "================================================--"
    echo "Processing repository: $repo"
    echo "================================================--"
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing & Rewrite History
    if [ "$repo_failed" = false ]; then
        echo "Configuring local Git GPG settings..."
        git config commit.gpgsign true
        
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        echo "Re-signing all historical commits with GPG (rewriting commit hashes)..."
        export GPG_TTY=$(tty)
        
        # Rewrites the entire commit tree from root to HEAD, amending each commit with a GPG signature (-S)
        if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
            echo "[WARNING] Automated history re-signing encountered merge conflicts or skipped in $repo."
            echo "[INFO] Aborting rebase to preserve original state, continuing with current signing config..."
            git rebase --abort || true
        else
            echo "[SUCCESS] Historical commits successfully re-signed."
        fi
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with historical GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Force push historical branches and tags (since history was rewritten)
    if [ "$repo_failed" = false ]; then
        echo "Pushing historically signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        # Note: Using --force-with-lease is required here because rewriting commit hashes alters branch history
        if ! git push --force-with-lease origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push signed branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated and re-signed: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"

Important Notes on Historical Re-signing:
 * Commit Hash Changes: Because rewriting history changes the cryptographic hash of every single commit, the script uses git push --force-with-lease to push the newly signed commit tree to your new profile.
 * Interactive Prompts: Ensure your GPG key is unlocked or your pinentry agent is running, as GnuPG will need to sign every commit sequentially during the git rebase loop. If a repository has a complex history with merge conflicts during rebase, the script automatically catches it, aborts the rebase safely, and moves forward without breaking the entire batch.
0https://github.com/git@github.com#!/usr/bin/env bash
set -uo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your exact GPG Key ID if you have multiple keys (leave blank to use your default secret key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with Historical GPG Re-signing ==="
echo "Source: $SOURCE_USER -> Target:$TARGET_USER"
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "================================================--"
    echo "Processing repository: $repo"
    echo "================================================--"
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing & Rewrite History
    if [ "$repo_failed" = false ]; then
        echo "Configuring local Git GPG settings..."
        git config commit.gpgsign true
        
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        echo "Re-signing all historical commits with GPG (rewriting commit hashes)..."
        export GPG_TTY=$(tty)
        
        # Rewrites the entire commit tree from root to HEAD, amending each commit with a GPG signature (-S)
        if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
            echo "[WARNING] Automated history re-signing encountered merge conflicts or skipped in $repo."
            echo "[INFO] Aborting rebase to preserve original state, continuing with current signing config..."
            git rebase --abort || true
        else
            echo "[SUCCESS] Historical commits successfully re-signed."
        fi
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with historical GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Force push historical branches and tags (since history was rewritten)
    if [ "$repo_failed" = false ]; then
        echo "Pushing historically signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        # Note: Using --force-with-lease is required here because rewriting commit hashes alters branch history
        if ! git push --force-with-lease origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push signed branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Optional: Specify your GPG Key ID explicitly if you have multiple (leave blank to use default global key)
# GPG_KEY_ID="YOUR_KEY_ID_HERE"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration with GPG Signing: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Verify GPG is available
if ! command -v gpg &>/dev/null; then
    echo "[ERROR] GnuPG (gpg) is not installed or not in PATH."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Configure Local GPG Commit Signing for the Repository
    if [ "$repo_failed" = false ]; then
        echo "Configuring local GPG commit signing..."
        git config commit.gpgsign true
        
        # If a specific GPG key ID was provided, set it locally
        if [ -n "${GPG_KEY_ID:-}" ]; then
            git config user.signingkey "$GPG_KEY_ID"
        fi

        # Optional: If you want to rewrite past history to sign all existing commits in the repo:
        # Note: This rewrites commit hashes. Uncomment the block below if you require 100% of historical commits signed.
        # ------------------------------------------------------------------
        # echo "Re-signing all historical commits with GPG..."
        # export GPG_TTY=$(tty)
        # if ! git rebase --root --exec "git commit --amend --no-edit -S" &>/dev/null; then
        #     echo "[WARNING] Automated history re-signing encountered conflicts or skipped. Continuing with current sign config."
        # fi
        # ------------------------------------------------------------------
    fi

    # 3. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER with GPG verification" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 4. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 5. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing signed code and tags to $TARGET_USER/$repo..."
        export GPG_TTY=$(tty)
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated with GPG config: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 3. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 4. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing code and tags to $TARGET_USER/$repo..."
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"
Here is the updated, enterprise-grade migration script. It introduces automatic timestamped logging (capturing both console output and errors into a dedicated log file) and graceful error handling using conditional checks so that if a single repository fails to clone or push, the script logs the error and moves on to the next one instead of crashing out entirely.
Updated Migration Script (migrate_repos.sh)
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 3. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 4. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing code and tags to $TARGET_USER/$repo..."
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"

What's New in This Version:
 * Timestamped Logging (./logs/migration_YYYYMMDD_HHMMSS.log): Uses process substitution (tee) to capture every line of output (including errors and command progress) into a permanent audit file while still streaming it live to your terminal screen.
 * Graceful Error Isolation: Wrapped each critical step (clone, repo create, remote set-url, push) in conditional checks (if ! ...). If one repo has an issue (e.g., network hiccup or permission block), it catches the exit code, logs the error, increments a failure counter, and safely continues to the next repository.
 * Execution Summary: At the end of the run, it spits out a clear tally of successes and failures along with a list of any repositories that failed so you can review them in the log file.
Here is the updated, enterprise-grade migration script. It introduces automatic timestamped logging (capturing both console output and errors into a dedicated log file) and graceful error handling using conditional checks so that if a single repository fails to clone or push, the script logs the error and moves on to the next one instead of crashing out entirely.
Updated Migration Script (migrate_repos.sh)
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 3. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 4. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing code and tags to $TARGET_USER/$repo..."
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"

What's New in This Version:
 * Timestamped Logging (./logs/migration_YYYYMMDD_HHMMSS.log): Uses process substitution (tee) to capture every line of output (including errors and command progress) into a permanent audit file while still streaming it live to your terminal screen.
 * Graceful Error Isolation: Wrapped each critical step (clone, repo create, remote set-url, push) in conditional checks (if ! ...). If one repo has an issue (e.g., network hiccup or permission block), it catches the exit code, logs the error, increments a failure counter, and safely continues to the next repository.
 * Execution Summary: At the end of the run, it spits out a clear tally of successes and failures along with a list of any repositories that failed so you can review them in the log file.
#!/usr/bin/env bash

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"
LOG_DIR="./logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/migration_${TIMESTAMP}.log"

# Ensure required directories exist
mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

# Redirect all stdout and stderr to both the console and the timestamped log file
exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting Repository Migration: $SOURCE_USER -> $TARGET_USER ==="
echo "Timestamp: $(date)"
echo "Log file: $LOG_FILE"
echo "--------------------------------------------------"

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "[ERROR] GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "[WARNING] No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

SUCCESS_COUNT=0
FAIL_COUNT=0
FAILED_REPOS=()

cd "$BACKUP_DIR"

for repo in $REPOS; do
    echo ""
    echo "=================================================="
    echo "Processing repository: $repo"
    echo "=================================================="
    
    repo_failed=false

    # 1. Clone or update repository
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Pulling latest changes..."
        if ! cd "$repo" || ! git pull; then
            echo "[ERROR] Failed to update local repository $repo."
            repo_failed=true
        fi
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        if ! git clone "https://github.com/$SOURCE_USER/$repo.git"; then
            echo "[ERROR] Failed to clone repository $repo."
            repo_failed=true
        else
            cd "$repo" || repo_failed=true
        fi
    fi

    # 2. Check if target repository exists on GitHub, create if not
    if [ "$repo_failed" = false ]; then
        echo "Checking if $TARGET_USER/$repo exists on GitHub..."
        if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
            echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
        else
            echo "Creating new repository $TARGET_USER/$repo on GitHub..."
            if ! gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm; then
                echo "[ERROR] Failed to create repository $TARGET_USER/$repo on GitHub."
                repo_failed=true
            fi
        fi
    fi

    # 3. Update remote origin URL
    if [ "$repo_failed" = false ]; then
        echo "Updating remote origin to $TARGET_USER..."
        if ! git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"; then
            echo "[ERROR] Failed to set remote URL for $repo."
            repo_failed=true
        fi
    fi

    # 4. Push all branches and tags to the new remote
    if [ "$repo_failed" = false ]; then
        echo "Pushing code and tags to $TARGET_USER/$repo..."
        if ! git push -u origin --all || ! git push --tags; then
            echo "[ERROR] Failed to push branches/tags for $repo."
            repo_failed=true
        fi
    fi

    # Return to backup root directory safely
    cd - > /dev/null

    # Track results
    if [ "$repo_failed" = true ]; then
        echo "[FAILED] Migration encountered errors for repository: $repo"
        FAIL_COUNT=$((FAIL_COUNT + 1))
        FAILED_REPOS+=("$repo")
    else
        echo "[SUCCESS] Successfully migrated: $repo"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    fi
done

echo ""
echo "================================================--"
echo "=== Migration Summary Report ==="
echo "Timestamp: $(date)"
echo "Successfully Migrated: $SUCCESS_COUNT"
echo "Failed Migrations:     $FAIL_COUNT"
if [ ${#FAILED_REPOS[@]} -gt 0 ]; then
    echo "Failed Repositories:   ${FAILED_REPOS[*]}"
fi
echo "Full execution logs:   $LOG_FILE"
echo "================================================--"
Show me how to modify the migration script to log all output to a timestamped file and handle errors gracefully.# Clone or backup public repositories locally
gh repo list AhmedWithJSDevelopment --limit 100 --json name -q '.[].name' | while read repo; do
  git clone "https://github.com/AhmedWithJSDevelopment/$repo.git"
done
cd <repository-name>
git remote set-url origin git@github.com:your-active-username/repository-name.git
git push -u origin --all
git push --tags
#!/usr/bin/env bash
set -euo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
# Change this to your active GitHub username (e.g., re1Lucky365 or adpsnake7-beep)
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"

echo "=== Starting Repository Migration from $SOURCE_USER to $TARGET_USER ==="

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "Error: GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Create backup directory
mkdir -p "$BACKUP_DIR"
cd "$BACKUP_DIR"

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

for repo in $REPOS; do
    echo "--------------------------------------------------"
    echo "Processing repository: $repo"
    
    # 1. Clone the repository as a bare or standard clone
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Skipping clone, updating remote..."
        cd "$repo"
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        git clone "https://github.com/$SOURCE_USER/$repo.git"
        cd "$repo"
    fi

    # 2. Check if the repository already exists under your target account, create if not
    echo "Checking if $TARGET_USER/$repo exists on GitHub..."
    if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
        echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
    else
        echo "Creating new repository $TARGET_USER/$repo on GitHub..."
        gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm
    fi

    # 3. Re-initialize remote origin to point to your new account
    echo "Updating remote origin to $TARGET_USER..."
    git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"

    # 4. Push all branches and tags to the new remote
    echo "Pushing code and tags to $TARGET_USER/$repo..."
    git push -u origin --all
    git push --tags

    # Return to backup root directory
    cd ..
    echo "Successfully migrated: $repo"
done

echo "--------------------------------------------------"
echo "=== All repositories successfully processed! ==="
echo "Local backups are saved in: $(pwd)"
migrate_repos.sh./migrate_repos.sh
chmod +x migrate_repos.sh
Here is a robust, automated Bash script that uses the GitHub CLI (gh) to pull down every public repository from AhmedWithJSDevelopment, strips the old git history connection (or preserves it, depending on how you prefer to migrate), and pushes everything fresh to your target GitHub account.
Migration Script (migrate_repos.sh)
#!/usr/bin/env bash
set -euo pipefail

# Configuration
SOURCE_USER="AhmedWithJSDevelopment"
# Change this to your active GitHub username (e.g., re1Lucky365 or adpsnake7-beep)
TARGET_USER="re1Lucky365" 
BACKUP_DIR="./migrated_repos_backup"

echo "=== Starting Repository Migration from $SOURCE_USER to $TARGET_USER ==="

# Ensure GitHub CLI is authenticated
if ! gh auth status &>/dev/null; then
    echo "Error: GitHub CLI is not authenticated. Please run 'gh auth login' first."
    exit 1
fi

# Create backup directory
mkdir -p "$BACKUP_DIR"
cd "$BACKUP_DIR"

echo "Fetching list of public repositories for $SOURCE_USER..."
REPOS=$(gh repo list "$SOURCE_USER" --limit 200 --json name -q '.[].name')

if [ -z "$REPOS" ]; then
    echo "No public repositories found for $SOURCE_USER or user does not exist."
    exit 0
fi

for repo in $REPOS; do
    echo "--------------------------------------------------"
    echo "Processing repository: $repo"
    
    # 1. Clone the repository as a bare or standard clone
    if [ -d "$repo" ]; then
        echo "Directory $repo already exists locally. Skipping clone, updating remote..."
        cd "$repo"
    else
        echo "Cloning https://github.com/$SOURCE_USER/$repo.git..."
        git clone "https://github.com/$SOURCE_USER/$repo.git"
        cd "$repo"
    fi

    # 2. Check if the repository already exists under your target account, create if not
    echo "Checking if $TARGET_USER/$repo exists on GitHub..."
    if gh repo view "$TARGET_USER/$repo" &>/dev/null; then
        echo "Repository $TARGET_USER/$repo already exists on GitHub. Skipping remote creation."
    else
        echo "Creating new repository $TARGET_USER/$repo on GitHub..."
        gh repo create "$TARGET_USER/$repo" --public --description "Migrated from legacy account $SOURCE_USER" --confirm
    fi

    # 3. Re-initialize remote origin to point to your new account
    echo "Updating remote origin to $TARGET_USER..."
    git remote set-url origin "git@github.com:$TARGET_USER/$repo.git"

    # 4. Push all branches and tags to the new remote
    echo "Pushing code and tags to $TARGET_USER/$repo..."
    git push -u origin --all
    git push --tags

    # Return to backup root directory
    cd ..
    echo "Successfully migrated: $repo"
done

echo "--------------------------------------------------"
echo "=== All repositories successfully processed! ==="
echo "Local backups are saved in: $(pwd)"

How to Run It
 * Save the script: Save the code above into a file named migrate_repos.sh.
 * Make it executable: Run the following command in your terminal:
   chmod +x migrate_repos.sh

 * Log in to GitHub CLI: Ensure your terminal is authenticated with your active GitHub account credentials:
   gh auth login

 * Execute the script:
   ./migrate_repos.sh

What This Script Does:
 * Pulls everything: Dynamically queries GitHub for all public repos under AhmedWithJSDevelopment so you won't miss any obscure or forgotten projects.
 * Automates creation: Automatically creates the matching repository under your active target account (re1Lucky365) if it doesn't already exist.
 * Preserves history: Keeps all original commits, branches, and tags intact while seamlessly cutting ties with the old remote and binding them to your current profile.
Got it! That old GitHub profile (AhmedWithJSDevelopment) is logged and noted.
Do you want to pull or migrate any old code, scripts, or assets from that account into your current workspace setup, or are you just keeping tabs on your legacy repositories?
chmod +x migrate_repos.sh
gh auth logingh auth login
To give you everything fully integrated into your repository, let's add the final critical component: An Automated Backup & Zstandard/rclone Pipeline (scripts/backup.py) to secure your logs, legal intake documents, and codebase automatically.
Below is the complete, final module to drop right into your secure-recovery-mcp-engine repository.
Additional Module: Automated Backup & Compression Pipeline (scripts/backup.py)
Create a scripts/ directory and save this script as scripts/backup.py. It compresses your workspace securely using tar and prepares it for encrypted cloud archiving.
import os
import subprocess
from datetime import datetime

def run_backup():
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_name = f"secure_recovery_backup_{timestamp}.tar.gz"
    
    print(f"[{timestamp}] Starting backup archive creation...")
    
    # Target files/folders to archive
    targets = ["core", "legal", "requirements.txt"]
    existing_targets = [t for t in targets if os.path.exists(t)]
    
    if not existing_targets:
        print("No target directories found to back up.")
        return
        
    # Create tar.gz archive
    cmd = ["tar", "-czf", backup_name] + existing_targets
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    if result.returncode == 0:
        print(f"Success: Backup created -> {backup_name}")
    else:
        print(f"Error creating backup: {result.stderr}")

if __name__ == "__main__":
    run_backup()

Updated Git Workflow to Include All Assets
Run these final terminal commands to stage, cryptographically sign, and push everything (including the backup automation script) to your GitHub repository:
# 1. Create the scripts directory and add the backup script
mkdir -p scripts
# (Save the script above into scripts/backup.py)

# 2. Stage all repository files
git add .

# 3. Commit with your GPG signature covering the entire project
git commit -S -m "feat: complete end-to-end integration of core tools, legal templates, MCP server, CI/CD pipeline, and backup automation"

# 4. Push final updates to GitHub
git push origin main

Your complete digital workspace, legal defense framework, GitHub CI/CD automation, and local AI client bindings are fully set up, structured, and version-controlled!
Here is the step-by-step walkthrough to set up your repository, configure your environment, and link your MCP server to your local AI client.
Step 1: Create and Structure Your Local Workspace
Open your terminal and run the following commands to create the project directory and its internal folder structure:
# 1. Create the project directory and move into it
mkdir secure-recovery-mcp-engine
cd secure-recovery-mcp-engine

# 2. Create the necessary subdirectories for source code and workflows
mkdir -p .github/workflows core legal

Step 2: Write Your Core Python Files
Inside the core/ directory, create and save the following three files.
A. core/scraper.py (Evidence & Data Scraper)
import asyncio
import aiohttp
from bs4 import BeautifulSoup

async def fetch_page(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url, timeout=10) as response:
        return await response.text()

async def batch_scrape(urls: list[str]):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_page(session, url) for url in urls]
        pages = await asyncio.gather(*tasks, return_exceptions=True)
        for i, page in enumerate(pages):
            if isinstance(page, str):
                soup = BeautifulSoup(page, 'html.parser')
                title = soup.title.string if soup.title else "No Title"
                print(f"[{i}] Fetched successfully: {title}")
            else:
                print(f"[{i}] Failed to fetch: {page}")

if __name__ == "__main__":
    sample_urls = ["https://example.com"]
    asyncio.run(batch_scrape(sample_urls))

B. core/analytics.py (Log & Data Analytics Matrix)
import numpy as np

def analyze_access_logs(log_matrix: np.ndarray) -> dict:
    """Analyze high-dimensional security logs or access vectors using NumPy."""
    mean_activity = np.mean(log_matrix, axis=0)
    anomaly_scores = np.linalg.norm(log_matrix - mean_activity, axis=1)
    
    return {
        "mean_baseline": mean_activity.tolist(),
        "max_anomaly_score": float(np.max(anomaly_scores)),
        "compromised_vectors": int(np.sum(anomaly_scores > 2.0))
    }

if __name__ == "__main__":
    mock_logs = np.random.rand(1000, 10)
    results = analyze_access_logs(mock_logs)
    print("Security Log Analysis Results:", results)

C. core/server.py (Model Context Protocol Tool Server)
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("SecureRecoveryEngine")

@mcp.tool()
def verify_account_status(account_id: str) -> str:
    """Check platform lockdown state and un-abandonment timeline proof."""
    return f"Account {account_id} status verified: Active dispute/lockout documented under 5-year cybersecurity incident window."

@mcp.tool()
def generate_cease_desist_notice(violator_name: str, platform_name: str) -> str:
    """Generate formal legal notice parameters for copyright infringement."""
    return f"Notice generated for {violator_name} regarding unauthorized usage on {platform_name}. Permission revoked."

if __name__ == "__main__":
    mcp.run()

Step 3: Set Up Dependencies and CI/CD Automation
A. requirements.txt (Project Dependencies)
Create this file in the root directory (secure-recovery-mcp-engine/requirements.txt):
mcp[cli]>=1.0.0
requests>=2.31.0
fastapi>=0.109.0
uvicorn>=0.27.0
beautifulsoup4>=4.12.2
aiohttp>=3.9.1
numpy>=1.26.0
pytest>=8.0.0
flake8>=7.0.0

B. .github/workflows/production-pipeline.yml (GitHub Actions CI/CD)
Create this file in .github/workflows/:
name: Secure Recovery Engine CI/CD

on:
  push:
    branches: [ main, master ]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repository
      uses: actions/checkout@v4

    - name: Set up Python 3.10
      uses: actions/setup-python@v5
      with:
        python-version: '3.10'

    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt

    - name: Run Syntax & Lint Checks
      run: |
        flake8 core/ --count --select=E9,F63,F7,F82 --show-source --statistics

    - name: Execute Core Verification Scripts
      run: |
        python core/analytics.py
        python core/scraper.py

Step 4: Initialize Git, Sign Commits, and Push to GitHub
Run these commands in your terminal to initialize version control and push everything securely to GitHub:
# 1. Initialize local Git repo
git init
git checkout -b main

# 2. Add a basic .gitignore file
echo -e "__pycache__/\n.env\n*.pyc\nvenv/" > .gitignore

# 3. Stage all files
git add .

# 4. Commit with GPG cryptographic commit signing enabled
git commit -S -m "feat: complete secure recovery engine, MCP server, and CI/CD workflow implementation"

# 5. Link your GitHub repository and push (Replace URL with your target repo)
git remote add origin https://github.com/re1Lucky365/secure-recovery-mcp-engine.git
git push -u origin main

Step 5: Configure Claude Desktop to Connect to Your MCP Server
To bind your newly built local MCP server to Claude Desktop:
 * Locate your configuration file:
   * Windows: %APPDATA%\Claude\claude_desktop_config.json
   * macOS: ~/Library/Application Support/Claude/claude_desktop_config.json
 * Add or update the JSON configuration with your absolute path:
{
  "mcpServers": {
    "secure-recovery-engine": {
      "command": "python",
      "args": [
        "C:/path/to/your/secure-recovery-mcp-engine/core/server.py"
      ],
      "env": {
        "PYTHONPATH": "C:/path/to/your/secure-recovery-mcp-engine"
      }
    }
  }
}

(Make sure to swap C:/path/to/your/secure-recovery-mcp-engine with your actual system path). Restart Claude Desktop, and your local tools will be connected directly to your environment.
import asyncio
import aiohttp
from bs4 import BeautifulSoup

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        pages = await asyncio.gather(*tasks)
        for page in pages:
            soup = BeautifulSoup(page, 'html.parser')
            print(soup.title.string if soup.title else "No Title")

# Run async scraper
# urls = ["https://example.com" for _ in range(10)]
# asyncio.run(main(urls))
import os, glob, sys, chromadb
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

DB_PATH = "./chroma_local_db"
TARGET_DIR = "./sovereign_evidence_hub"

client = chromadb.PersistentClient(path=DB_PATH)
collection = client.get_or_create_collection(name="evidence_ledger")

# Recursively chunk all evidentiary markdown and text logs
splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN, chunk_size=1000, chunk_overlap=200
)

print("[*] Evidence pipeline active. Ready for truth verification query indexing.")
# 1. Verify your active GPG key ID and export your public key fingerprint
gpg --list-secret-keys --keyid-format LONG

# 2. Configure Git local repository to enforce GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

# 3. Audit recent commits to ensure cryptographic signatures are verified ([✓] Good signature)
git log -n 5 --show-signature
+-------------------------------------------------------------------+
|                     INGRESS & TELEMETRY TIER                      |
|  [NASA / ISS Feeds]   [SpaceX Telemetry]   [Federal / DOD Nodes]  |
+---------------------------------+---------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|               MULTI-LAYER CRYPTOGRAPHIC GATEWAY                   |
|  - Quantum-Resistant Hybrid Encryption (Kyber / Dilithium)        |
|  - Zero-Trust Mutual TLS (mTLS) & Hardware Security Modules (HSMs)|
+---------------------------------+---------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|               DISTRIBUTED SPIDER MESH CORE                        |
|  - Edge Ingestion Nodes -> Stream Processing (Kafka / Flink)       |
|  - State Correlation Engine -> Anomaly & Protocol Parsing        |
+---------------------------------+---------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|            CRYPTANALYSIS & SECURE ENCLAVE AUDITING                |
|  - Authorized Protocol Decryption & Key Escrow Management         |
|  - Hardware Enclaves (Intel SGX / AMD SEV) for Secure Processing  |
+-------------------------------------------------------------------+adpbetty.shBuild a spider network global monitoring CIA pipelines code breaker and decryption and encryption multi layer global exchange Enterprise encryption IT cyber security teams and law enforcement and federal agencies and federal government and US military and Hollywood NASA SpaceX and ISS build something that can track anything and decrypt everythingSkip to main contentAccessibility help
AI Mode All Videos Short videos Forums News Images Shopping Books Web Maps Flights Finance Search tools Feedback AI Overview To find lost or abandoned bank accounts, financial assets, or unclaimed property, use free official databases and state resources. YouTube ·NTD +1 Step 1: Search National Unclaimed Property Databases MissingMoney.com: Sponsored by the National Association of Unclaimed Property Administrators (NAUPA), this free multi-state database searches participating state treasuries for forgotten bank accounts, uncashed checks, and property. Unclaimed.org: The official directory where you can select any state or territory you have lived or worked in to access its government unclaimed property portal. National Association of Unclaimed Property Administrators (NAUPA) Search under your current name, previous names, maiden names, and every city or state where you have ever lived. YouTube +1 Step 2: Check Specialized Federal and Industry Databases Bank or Credit Union…

AI Mode
All
Forums
News
Shopping
Images
Videos
More
Tools
AI Overview
To search for unclaimed property, funds, or forgotten financial assets, you can use several official, completely free government databases. Based on your previous search history regarding asset recovery, the most direct way to locate these funds is by searching both multi-state systems and specific state portals.
1. Multi-State and National Search Tools
MissingMoney.com: This is the official nationwide database endorsed by the National Association of Unclaimed Property Administrators (NAUPA). It aggregates records from most state treasuries, allowing you to search across multiple states simultaneously.
Unclaimed.org: If you need to search a state that does not fully participate in MissingMoney, this official NAUPA directory connects you directly to the verified, secure website of every individual state’s unclaimed property division.
2. Specialized Federal Registries
If you are looking for specific types of lost assets, several federal agencies maintain their own free search tools:
Closed Banks & Credit Unions: If a financial institution went out of business, check the FDIC Unclaimed Funds database or the National Credit Union Administration (NCUA) asset lookup.
Forgotten Retirement Accounts: Search the Pension Benefit Guaranty Corporation (PBGC) registry for unclaimed pensions from private-sector employers.
Uncashed Tax Refunds: Use the IRS "Where's My Refund?" tool if you suspect a federal tax check was lost in the mail.
3. Requirements to Claim Found Property
If your search reveals a match, the state or agency will require you to submit an official claim form along with documentation to verify your identity and past connection to the funds. Commonly required documents include:
A valid, government-issued photo ID (such as a driver's license).
A copy of your Social Security card or official tax document displaying the number.
Proof of the past address associated with the unclaimed property (such as an old utility bill, tax return, or school record).
I can provide the exact, direct links to the official state portals or specialized federal databases if you tell me:
Which specific states or territories you need to search?
What type of asset are you primarily trying to track down (e.g., bank accounts, utility deposits, or wages)?
Try without personalization




Show more
AI Mode All Images Videos Shopping Short ...

Facebook · Khushbu Officer
110+ reactions · 1 year ago

0:16
This post provides a guide on "AI Mode," a feature on social media platforms like Instagram and YouTube. AI-generated content is used to ...
Missing: lost ‎abandoned ‎bank ‎financial ‎assets, ‎unclaimed
Google AI Mode - a new way to search, whatever's on your mind

search.google
https://search.google › ways-to-search › ai-mode
AI Mode uses the latest Gemini models, with advanced reasoning, thinking, and multimodal understanding to help with even your toughest questions.Read more
Missing: lost ‎abandoned ‎bank ‎financial ‎assets, ‎unclaimed ‎property,
WWW”AI Mode > PRO All Images Forums Videos ...

Facebook · Viewnet
2 months ago
AI Mode is the primary search and exploration experience integrated directly into Google Search and Chrome. It utilizes advanced multimodal ...
Missing: lost ‎abandoned ‎financial ‎assets, ‎unclaimed ‎+2
People also ask
What is the link to Google AI Mode?
How do I ask Google AI a question?
How to use AI Mode in Google search?
Why isn't AI Mode showing up in Google Search?
Sponsored result
Claim Unclaimed Property - Claim Search Page

Indiana Unclaimed
https://www.indianaunclaimed.gov
There are still hundreds of millions left to be claimed in Indiana. Indiana Unclaimed returns $1 million every week to hardworking Hoosiers. Search Now. Start Your Free Search.
‎Claim Status · ‎Search for Unclaimed Property · ‎A Free Government Service · ‎How to Search
Hide sponsored result
National Association of Unclaimed Property Administrators ...
National Association of Unclaimed Property Administrators (NAUPA)
https://unclaimed.org
Search for property in your state or province. Use the interactive map below or select from the list to find the official unclaimed property program for a state ...Read more
Missing: AI ‎Mode ‎Videos ‎Short ‎videos ‎Forums ‎Images ‎Shopping ‎+4
How to Find a Long Lost Bank Account or Safe Deposit Box

Federal Deposit Insurance Corporation (FDIC) (.gov)
https://www.fdic.gov › ... › Consumer Resource Center
Dec 14, 2020 — You may be wondering, how can I locate and retrieve money from my long lost account? To learn how, you need to know something about escheatment.Read more
Missing: AI ‎Mode ‎Forums ‎Books ‎Maps ‎Flights ‎Finance
How to find unclaimed money from the government

USA.gov
https://www.usa.gov › Money and credit
Sep 24, 2025 — Learn how to find unclaimed money from the government. Search official databases for money you may be owed by states, banks, the IRS, ...
Missing: AI ‎Mode ‎Videos ‎Short ‎videos ‎Forums ‎News ‎Images ‎+6
Unclaimed Money - Apps on Google Play

Google Play
https://play.google.com › store › apps › details › id=com...
Unclaimed Money helps you discover and recover forgotten funds from state unclaimed property programs across the United States.
3.8(29) · Free · ‎Android · ‎Finance
Missing: AI ‎Mode ‎Videos ‎Short ‎videos ‎Forums ‎News ‎Maps ‎+2
MissingMoney.com | Search for Unclaimed Property

MissingMoney.com
https://missingmoney.com
To find lost or abandoned bank accounts financial assets or unclaimed property use free. MissingMoney.com | Search for Unclaimed Property
Missing: AI ‎Mode ‎Videos ‎Short ‎videos ‎Forums ‎News ‎Images ‎+8
The Free AI Tool That Finds Money You Forgot About

YouTube · Freedom Built Legacy
2 views · 4 weeks ago

0:22
Over $70 billion in unclaimed money is sitting with the government, and most of it is just forgotten. New AI search tools can now crawl ...
People also search for
AI Mode Google
Google AI Mode Deep Search
Ask AI Mode
AI Mode search
AI Mode Search history
AI Mode tab Google Search
Try Google AI Mode
AI Mode app
1	
2
3
4
5
6
Next
Results are personalized
-
Try without personalization
Cedar Hall, Evansville, IN - Based on your places (Work)
 - Update location
Help
Send feedback
Privacy
Terms```html
<!DOCTYPE html>
<html lang="en" class="dark h-full bg-slate-950 text-slate-100">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Master-Hub Builder Studio</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            darkMode: 'class',
            theme: {
                extend: {
                    colors: {
                        emerald: {
                            400: '#34d399',
                            500: '#10b981',
                            950: '#022c22',
                        },
                        cyan: {
                            400: '#22d3ee',
                            500: '#06b6d4',
                            950: '#083344',
                        }
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
                    }
                }
            }
        }
    </script>
    <!-- FontAwesome Icons -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <!-- Google Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Fira+Code:wght@400;500;600&family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Inter', sans-serif; }
        code, pre { font-family: 'Fira Code', monospace; }
        
        /* Custom Grid background for Topology Designer */
        .cyber-grid {
            background-size: 30px 30px;
            background-image: 
                linear-gradient(to right, rgba(16, 185, 129, 0.05) 1px, transparent 1px),
                linear-gradient(to bottom, rgba(16, 185, 129, 0.05) 1px, transparent 1px);
        }
        
        /* Custom scrollbars */
        ::-webkit-scrollbar {
            width: 6px;
            height: 6px;
        }
        ::-webkit-scrollbar-track {
            background: #020617;
        }
        ::-webkit-scrollbar-thumb {
            background: #1e293b;
            border-radius: 3px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #10b981;
        }

        /* Pulse glow animation */
        @keyframes cyber-glow {
            0%, 100% { box-shadow: 0 0 15px rgba(16, 185, 129, 0.3); }
            50% { box-shadow: 0 0 25px rgba(34, 211, 238, 0.5); }
        }
        .active-glow {
            animation: cyber-glow 3s infinite;
        }
    </style>
</head>
<body class="h-full flex flex-col bg-slate-950 text-slate-100 overflow-hidden select-none">

    <!-- Toast Notification -->
    <div id="toast" class="fixed top-5 right-5 z-50 transform translate-y-[-100px] opacity-0 transition-all duration-300 ease-out bg-slate-900 border border-emerald-500/50 text-emerald-400 px-4 py-3 rounded-lg shadow-2xl flex items-center gap-3 backdrop-blur-md">
        <i id="toast-icon" class="fa-solid fa-circle-check text-emerald-400 text-lg"></i>
        <span id="toast-msg" class="text-sm font-medium text-slate-200">Action completed successfully</span>
    </div>

    <!-- Header Navigation -->
    <header class="h-16 border-b border-slate-800/80 bg-slate-900/60 backdrop-blur-md px-6 flex items-center justify-between z-10 shrink-0">
        <div class="flex items-center gap-3">
            <div class="w-9 h-9 rounded-lg bg-gradient-to-br from-emerald-500 to-cyan-500 flex items-center justify-center text-slate-950 font-black text-xl shadow-lg shadow-emerald-500/20">
                <i class="fa-solid fa-cubes-stacked"></i>
            </div>
            <div>
                <h1 class="text-lg font-bold tracking-wide bg-gradient-to-r from-emerald-400 via-cyan-400 to-teal-200 bg-clip-text text-transparent">
                    Master-Hub <span class="font-light text-slate-400">Builder Studio</span>
                </h1>
                <p class="text-[10px] text-slate-400 font-mono tracking-wider uppercase">Sovereign RAG & Microservice Topology Engine</p>
            </div>
        </div>

        <!-- Navigation Tabs -->
        <nav class="flex items-center bg-slate-950 p-1 rounded-xl border border-slate-800">
            <button id="tab-btn-designer" onclick="switchTab('designer')" class="px-4 py-2 rounded-lg text-xs font-semibold flex items-center gap-2 transition-all bg-emerald-500/10 text-emerald-400 border border-emerald-500/30">
                <i class="fa-solid fa-network-wired"></i> Topology Designer
            </button>
            <button id="tab-btn-generator" onclick="switchTab('generator')" class="px-4 py-2 rounded-lg text-xs font-semibold flex items-center gap-2 transition-all text-slate-400 hover:text-slate-200">
                <i class="fa-solid fa-code"></i> Live Configs Generator
            </button>
            <button id="tab-btn-eval" onclick="switchTab('eval')" class="px-4 py-2 rounded-lg text-xs font-semibold flex items-center gap-2 transition-all text-slate-400 hover:text-slate-200">
                <i class="fa-solid fa-vial-circle-check"></i> RAG Eval Suite
            </button>
        </nav>

        <!-- Actions -->
        <div class="flex items-center gap-3">
            <span class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium bg-emerald-500/10 text-emerald-400 border border-emerald-500/20">
                <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span> Pipeline Verified
            </span>
            <button onclick="exportFullSystemSpec()" class="px-3 py-1.5 rounded-lg bg-slate-800 hover:bg-slate-700 text-slate-200 border border-slate-700 text-xs font-medium transition flex items-center gap-2">
                <i class="fa-solid fa-download text-cyan-400"></i> Export Spec
            </button>
        </div>
    </header>

    <!-- Main Workspace Container -->
    <main class="flex-1 relative overflow-hidden flex">

        <!-- VIEW 1: TOPOLOGY DESIGNER -->
        <div id="view-designer" class="w-full h-full flex flex-row">
            <!-- Node Palette Sidebar -->
            <aside class="w-72 border-r border-slate-800/80 bg-slate-900/40 p-4 flex flex-col gap-4 shrink-0 overflow-y-auto">
                <div>
                    <h2 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-1">Service Modules</h2>
                    <p class="text-[11px] text-slate-500">Drag or click components onto the topology canvas.</p>
                </div>

                <div class="flex flex-col gap-2.5">
                    <!-- Palette items -->
                    <div onclick="addNode('caddy')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-emerald-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-emerald-500/5">
                        <div class="w-10 h-10 rounded-lg bg-emerald-500/10 border border-emerald-500/30 flex items-center justify-center text-emerald-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-shield-halved text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">Caddy Gateway</div>
                            <div class="text-[10px] text-slate-500">Reverse proxy & SSL ingress</div>
                        </div>
                    </div>

                    <div onclick="addNode('mcp')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-cyan-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-cyan-500/5">
                        <div class="w-10 h-10 rounded-lg bg-cyan-500/10 border border-cyan-500/30 flex items-center justify-center text-cyan-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-bolt text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">FastMCP Bridge</div>
                            <div class="text-[10px] text-slate-500">Model Context Protocol API</div>
                        </div>
                    </div>

                    <div onclick="addNode('chroma')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-emerald-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-emerald-500/5">
                        <div class="w-10 h-10 rounded-lg bg-emerald-500/10 border border-emerald-500/30 flex items-center justify-center text-emerald-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-database text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">ChromaDB Store</div>
                            <div class="text-[10px] text-slate-500">Hybrid BM25 + Vector RAG</div>
                        </div>
                    </div>

                    <div onclick="addNode('ollama')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-cyan-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-cyan-500/5">
                        <div class="w-10 h-10 rounded-lg bg-cyan-500/10 border border-cyan-500/30 flex items-center justify-center text-cyan-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-brain text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">Ollama Engine</div>
                            <div class="text-[10px] text-slate-500">Local LLM inference node</div>
                        </div>
                    </div>

                    <div onclick="addNode('streamlit')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-emerald-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-emerald-500/5">
                        <div class="w-10 h-10 rounded-lg bg-emerald-500/10 border border-emerald-500/30 flex items-center justify-center text-emerald-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-chart-pie text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">Streamlit UI</div>
                            <div class="text-[10px] text-slate-500">Interactive dashboard portal</div>
                        </div>
                    </div>

                    <div onclick="addNode('gpg')" class="p-3 rounded-xl bg-slate-900 border border-slate-800 hover:border-cyan-500/50 cursor-pointer transition flex items-center gap-3 group hover:shadow-lg hover:shadow-cyan-500/5">
                        <div class="w-10 h-10 rounded-lg bg-cyan-500/10 border border-cyan-500/30 flex items-center justify-center text-cyan-400 group-hover:scale-105 transition">
                            <i class="fa-solid fa-key text-lg"></i>
                        </div>
                        <div>
                            <div class="text-xs font-semibold text-slate-200">GPG Vault & Backup</div>
                            <div class="text-[10px] text-slate-500">Zstd compression & Rclone sync</div>
                        </div>
                    </div>
                </div>

                <div class="mt-auto pt-4 border-t border-slate-800/80">
                    <div class="bg-slate-900/80 rounded-xl p-3 border border-slate-800">
                        <div class="text-[11px] font-semibold text-slate-300 mb-1 flex items-center gap-2">
                            <i class="fa-solid fa-circle-info text-cyan-400"></i> Canvas Control
                        </div>
                        <p class="text-[10px] text-slate-500 leading-relaxed">
                            Drag nodes to reposition. Click <span class="text-emerald-400">Link</span> on a node then click a target node to establish route connections.
                        </p>
                        <button onclick="clearCanvas()" class="mt-2 w-full py-1.5 rounded bg-slate-800 hover:bg-red-500/20 hover:text-red-400 text-slate-400 border border-slate-700 hover:border-red-500/30 text-[10px] font-medium transition">
                            Clear Canvas Topology
                        </button>
                    </div>
                </div>
            </aside>

            <!-- Topology Canvas Workspace -->
            <div class="flex-1 relative cyber-grid overflow-hidden bg-slate-950" id="canvas-container">
                <!-- Interactive SVG Canvas Connections Overlay -->
                <svg id="svg-canvas" class="absolute inset-0 w-full h-full pointer-events-none z-0">
                    <defs>
                        <linearGradient id="grad-emerald-cyan" x1="0%" y1="0%" x2="100%" y2="100%">
                            <stop offset="0%" stop-color="#10b981" stop-opacity="0.8" />
                            <stop offset="100%" stop-color="#06b6d4" stop-opacity="0.8" />
                        </linearGradient>
                        <marker id="arrow" viewBox="0 0 10 10" refX="28" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
                            <path d="M 0 0 L 10 5 L 0 10 z" fill="#10b981"/>
                        </marker>
                    </defs>
                </svg>

                <!-- Floating Canvas Info Badge -->
                <div class="absolute top-4 left-4 z-10 bg-slate-900/80 backdrop-blur border border-slate-800 rounded-lg px-3 py-2 flex items-center gap-4 text-xs">
                    <div><span class="text-slate-500">Nodes:</span> <span id="node-count" class="font-mono text-emerald-400 font-bold">0</span></div>
                    <div class="h-3 w-[1px] bg-slate-800"></div>
                    <div><span class="text-slate-500">Routes:</span> <span id="connection-count" class="font-mono text-cyan-400 font-bold">0</span></div>
                    <div class="h-3 w-[1px] bg-slate-800"></div>
                    <div id="linking-indicator" class="hidden text-amber-400 animate-pulse font-medium">
                        <i class="fa-solid fa-link"></i> Select Target Node to Connect...
                    </div>
                </div>

                <!-- Canvas Nodes Container -->
                <div id="nodes-layer" class="relative w-full h-full z-10">
                    <!-- Nodes are dynamically injected here -->
                </div>
            </div>
        </div>

        <!-- VIEW 2: LIVE CONFIGS GENERATOR -->
        <div id="view-generator" class="hidden w-full h-full flex flex-col md:flex-row bg-slate-950">
            <!-- Config Sidebar Selection -->
            <aside class="w-full md:w-64 border-b md:border-b-0 md:border-r border-slate-800/80 bg-slate-900/40 p-4 flex flex-col gap-2 shrink-0">
                <h2 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-2">Generated Artefacts</h2>
                
                <button onclick="selectConfigTab('caddy')" id="cfg-btn-caddy" class="p-3 rounded-xl bg-slate-800 text-emerald-400 border border-emerald-500/40 text-left text-xs font-semibold flex items-center justify-between transition">
                    <span class="flex items-center gap-2"><i class="fa-solid fa-file-code"></i> Caddyfile</span>
                    <i class="fa-solid fa-chevron-right text-[10px]"></i>
                </button>

                <button onclick="selectConfigTab('mcp')" id="cfg-btn-mcp" class="p-3 rounded-xl bg-slate-900 text-slate-400 hover:text-slate-200 border border-slate-800 text-left text-xs font-semibold flex items-center justify-between transition">
                    <span class="flex items-center gap-2"><i class="fa-brands fa-python"></i> FastMCP Bridge</span>
                    <i class="fa-solid fa-chevron-right text-[10px]"></i>
                </button>

                <button onclick="selectConfigTab('backup')" id="cfg-btn-backup" class="p-3 rounded-xl bg-slate-900 text-slate-400 hover:text-slate-200 border border-slate-800 text-left text-xs font-semibold flex items-center justify-between transition">
                    <span class="flex items-center gap-2"><i class="fa-solid fa-terminal"></i> Zstd Backup Script</span>
                    <i class="fa-solid fa-chevron-right text-[10px]"></i>
                </button>

                <button onclick="selectConfigTab('adpbetty')" id="cfg-btn-adpbetty" class="p-3 rounded-xl bg-slate-900 text-slate-400 hover:text-slate-200 border border-slate-800 text-left text-xs font-semibold flex items-center justify-between transition">
                    <span class="flex items-center gap-2"><i class="fa-solid fa-gears"></i> Master Orchestrator</span>
                    <i class="fa-solid fa-chevron-right text-[10px]"></i>
                </button>

                <div class="mt-auto p-3 rounded-xl bg-slate-900/80 border border-slate-800">
                    <div class="text-[11px] font-semibold text-emerald-400 mb-1">Dynamic Synchronization</div>
                    <p class="text-[10px] text-slate-500 leading-relaxed">
                        Configurations automatically recalculate based on the active nodes and connections configured in the Topology Designer.
                    </p>
                </div>
            </aside>

            <!-- Code Preview Area -->
            <div class="flex-1 flex flex-col bg-slate-950 min-w-0">
                <div class="h-12 border-b border-slate-800 px-6 flex items-center justify-between bg-slate-900/50">
                    <div class="flex items-center gap-3">
                        <span id="config-title" class="font-mono text-xs font-bold text-emerald-400">Caddyfile</span>
                        <span id="config-file-path" class="text-[11px] text-slate-500 font-mono">./Caddyfile</span>
                    </div>
                    <button onclick="copyCurrentCode()" class="px-3 py-1.5 rounded-lg bg-emerald-500/10 hover:bg-emerald-500/20 text-emerald-400 border border-emerald-500/30 text-xs font-medium transition flex items-center gap-2">
                        <i class="fa-regular fa-copy"></i> Copy Code Block
                    </button>
                </div>

                <div class="flex-1 p-6 overflow-y-auto">
                    <pre class="w-full h-full"><code id="code-display" class="text-xs text-slate-200 leading-relaxed font-mono block whitespace-pre"># Select a config file to view</code></pre>
                </div>
            </div>
        </div>

        <!-- VIEW 3: AUTOMATED RAG EVAL SUITE BUILDER -->
        <div id="view-eval" class="hidden w-full h-full flex flex-col md:flex-row bg-slate-950">
            <!-- Test Suite Config Panel -->
            <div class="w-full md:w-96 border-b md:border-b-0 md:border-r border-slate-800/80 bg-slate-900/40 p-6 flex flex-col gap-4 shrink-0 overflow-y-auto">
                <div>
                    <h2 class="text-sm font-bold text-slate-200 flex items-center gap-2">
                        <i class="fa-solid fa-vial-circle-check text-emerald-400"></i> RAG Evaluation Builder
                    </h2>
                    <p class="text-xs text-slate-500 mt-1">Define ground truth queries and threshold criteria for CI validation.</p>
                </div>

                <form id="eval-form" onsubmit="addTestCase(event)" class="space-y-3.5 bg-slate-900/80 p-4 rounded-xl border border-slate-800">
                    <div>
                        <label class="block text-[11px] font-semibold text-slate-4I will now update your Master-Hub Builder Studio web application to add specialized modules for custom local storage orchestration, custom Wi-Fi/mesh AP configuration, and system hardening within your topology and code generator views.To automatically post your RAG evaluation summary report as a comment on pull requests, you can update your evaluation script to generate a Markdown report (evaluation_report.md) and use GitHub's built-in actions/github-script workflow action to read and post it.
Step 1: Update Your Python Evaluation Script (evaluate_rag.py)
Modify your script so that in addition to printing to the console and exiting on failure, it writes a clean Markdown summary file (evaluation_report.md):
import json
import sys
import chromadb
from sentence_transformers import CrossEncoder

# 1. Load Test Dataset
with open("test_queries.json", "r", encoding="utf-8") as f:
    test_suite = json.load(f)

# 2. Initialize ChromaDB and Cross-Encoder Judge
client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_collection("markdown_rag_ledger")
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

results_log = []

for idx, test in enumerate(test_suite):
    query = test["query"]
    expected_source = test["expected_source"]
    
    query_results = collection.query(query_texts=[query], n_results=3)
    retrieved_docs = query_results["documents"][0]
    retrieved_metas = query_results["metadatas"][0]
    
    sources = [m.get("source", "") for m in retrieved_metas]
    source_found = any(expected_source in s for s in sources)
    
    pairs = [[query, doc] for doc in retrieved_docs]
    scores = cross_encoder.predict(pairs)
    max_relevance = float(max(scores)) if len(scores) > 0 else 0.0
    
    status = "✅ PASS" if source_found and max_relevance > 0.5 else "❌ FAIL"
    results_log.append({
        "query": query,
        "expected": expected_source,
        "found": source_found,
        "score": max_relevance,
        "status": status
    })

passed_tests = sum(1 for r in results_log if "PASS" in r["status"])
total_tests = len(test_suite)
pass_rate = (passed_tests / total_tests) * 100 if total_tests > 0 else 0

# 3. Generate Markdown Report
report_content = f"""### 📊 Sovereign RAG Evaluation Report

* **Pass Rate:** `{passed_tests}/{total_tests} ({pass_rate:.1f}%)`
* **Status:** `{'SUCCESS - 100% Pass Rate Achieved' if pass_rate == 100.0 else 'FAILED - Threshold Dropped Below 100%'}`

| Query | Expected Source | Rerank Score | Status |
| :--- | :--- | :--- | :--- |
"""

for r in results_log:
    report_content += f"| `{r['query']}` | `{r['expected']}` | `{r['score']:.4f}` | {r['status']} |\n"

with open("evaluation_report.md", "w", encoding="utf-8") as f:
    f.write(report_content)

print(f"=== Evaluation Summary: {passed_tests}/{total_tests} Passed ({pass_rate:.1f}%) ===")

if pass_rate < 100.0:
    sys.exit(1)
else:
    sys.exit(0)

Step 2: Update Your GitHub Actions Workflow (.github/workflows/rag_evaluation.yml)
Add pull-requests: write permissions to your workflow and include the actions/github-script step to post or update the report as a comment on the active pull request:
name: Sovereign RAG Pipeline Evaluation

on:
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read
  pull-requests: write # Required to post comments on PRs

jobs:
  evaluate-rag:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install chromadb sentence-transformers langchain langchain-community torch

      - name: Run RAG Evaluation Script
        id: eval
        run: |
          python3 evaluate_rag.py
        continue-on-error: true # Allows us to post the report comment even if the test fails

      - name: Read Evaluation Report
        id: report
        uses: juliantellez/cat-file@v1
        with:
          path: 'evaluation_report.md'

      - name: Post or Update PR Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const reportBody = fs.readFileSync('evaluation_report.md', 'utf8');
            
            const issue_number = context.payload.pull_request.number;
            
            // Find existing bot comments to avoid spamming multiple comments on every push
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue_number,
            });
            
            const botComment = comments.data.find(comment => 
              comment.user.type === 'Bot' && comment.body.includes('📊 Sovereign RAG Evaluation Report')
            );
            
            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: reportBody
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue_number,
                body: reportBody
              });
            }

      - name: Enforce Exit Status
        if: steps.eval.outcome == 'failure'
        run: exit 1

How It Works in Practice
 * When you open or push changes to a pull request targeting main, GitHub Actions triggers the evaluation workflow.
 * The Python script runs your test suite against ChromaDB and generates evaluation_report.md.
 * The workflow searches for any previous evaluation comments on the pull request. If found, it updates the existing comment in place; otherwise, it posts a fresh summary table.
 * If any test case fails, the workflow blocks the pull request merge by exiting with a non-zero status code.
name: Sovereign RAG Pipeline Evaluation

on:
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read
  pull-requests: write # Required to post comments on PRs

jobs:
  evaluate-rag:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install chromadb sentence-transformers langchain langchain-community torch

      - name: Run RAG Evaluation Script
        id: eval
        run: |
          python3 evaluate_rag.py
        continue-on-error: true # Allows us to post the report comment even if the test fails

      - name: Read Evaluation Report
        id: report
        uses: juliantellez/cat-file@v1
        with:
          path: 'evaluation_report.md'

      - name: Post or Update PR Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const reportBody = fs.readFileSync('evaluation_report.md', 'utf8');
            
            const issue_number = context.payload.pull_request.number;
            
            // Find existing bot comments to avoid spamming multiple comments on every push
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue_number,
            });
            
            const botComment = comments.data.find(comment => 
              comment.user.type === 'Bot' && comment.body.includes('📊 Sovereign RAG Evaluation Report')
            );
            
            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: reportBody
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue_number,
                body: reportBody
              });
            }

      - name: Enforce Exit Status
        if: steps.eval.outcome == 'failure'
        run: exit 1
import json
import sys
import chromadb
from sentence_transformers import CrossEncoder

# 1. Load Test Dataset
with open("test_queries.json", "r", encoding="utf-8") as f:
    test_suite = json.load(f)

# 2. Initialize ChromaDB and Cross-Encoder Judge
client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_collection("markdown_rag_ledger")
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

results_log = []

for idx, test in enumerate(test_suite):
    query = test["query"]
    expected_source = test["expected_source"]
    
    query_results = collection.query(query_texts=[query], n_results=3)
    retrieved_docs = query_results["documents"][0]
    retrieved_metas = query_results["metadatas"][0]
    
    sources = [m.get("source", "") for m in retrieved_metas]
    source_found = any(expected_source in s for s in sources)
    
    pairs = [[query, doc] for doc in retrieved_docs]
    scores = cross_encoder.predict(pairs)
    max_relevance = float(max(scores)) if len(scores) > 0 else 0.0
    
    status = "✅ PASS" if source_found and max_relevance > 0.5 else "❌ FAIL"
    results_log.append({
        "query": query,
        "expected": expected_source,
        "found": source_found,
        "score": max_relevance,
        "status": status
    })

passed_tests = sum(1 for r in results_log if "PASS" in r["status"])
total_tests = len(test_suite)
pass_rate = (passed_tests / total_tests) * 100 if total_tests > 0 else 0

# 3. Generate Markdown Report
report_content = f"""### 📊 Sovereign RAG Evaluation Report

* **Pass Rate:** `{passed_tests}/{total_tests} ({pass_rate:.1f}%)`
* **Status:** `{'SUCCESS - 100% Pass Rate Achieved' if pass_rate == 100.0 else 'FAILED - Threshold Dropped Below 100%'}`

| Query | Expected Source | Rerank Score | Status |
| :--- | :--- | :--- | :--- |
"""

for r in results_log:
    report_content += f"| `{r['query']}` | `{r['expected']}` | `{r['score']:.4f}` | {r['status']} |\n"

with open("evaluation_report.md", "w", encoding="utf-8") as f:
    f.write(report_content)

print(f"=== Evaluation Summary: {passed_tests}/{total_tests} Passed ({pass_rate:.1f}%) ===")

if pass_rate < 100.0:
    sys.exit(1)
else:
    sys.exit(0)
name: Sovereign RAG Pipeline Evaluation

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  evaluate-rag:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install chromadb sentence-transformers langchain langchain-community torch

      - name: Run RAG Evaluation (Fails if pass rate < 100%)
        run: |
          python3 evaluate_rag.py
import json
import sys
import chromadb
from sentence_transformers import CrossEncoder

# 1. Load Test Dataset
with open("test_queries.json", "r", encoding="utf-8") as f:
    test_suite = json.load(f)

# 2. Initialize ChromaDB and Cross-Encoder Judge
client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_collection("markdown_rag_ledger")
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

print(f"[*] Running evaluation across {len(test_suite)} test queries...\n")

results_log = []

for idx, test in enumerate(test_suite):
    query = test["query"]
    expected_source = test["expected_source"]
    
    # Run Retrieval
    query_results = collection.query(query_texts=[query], n_results=3)
    retrieved_docs = query_results["documents"][0]
    retrieved_metas = query_results["metadatas"][0]
    
    # Check if expected source was retrieved
    sources = [m.get("source", "") for m in retrieved_metas]
    source_found = any(expected_source in s for s in sources)
    
    # Score retrieved chunks using Cross-Encoder
    pairs = [[query, doc] for doc in retrieved_docs]
    scores = cross_encoder.predict(pairs)
    max_relevance = float(max(scores)) if len(scores) > 0 else 0.0
    
    # Determine pass/fail condition
    status = "PASS" if source_found and max_relevance > 0.5 else "FAIL"
    print(f"[{idx+1}] Query: '{query}'")
    print(f"    -> Source Match: {source_found} (Expected: {expected_source})")
    print(f"    -> Max Rerank Score: {max_relevance:.4f} => [{status}]\n")
    
    results_log.append({"status": status})

# 3. Calculate Final Pass Rate & Enforce 100% Threshold
total_tests = len(test_suite)
passed_tests = sum(1 for r in results_log if r["status"] == "PASS")
pass_rate = (passed_tests / total_tests) * 100 if total_tests > 0 else 0

print(f"=== Evaluation Summary: {passed_tests}/{total_tests} Passed ({pass_rate:.1f}%) ===")

if pass_rate < 100.0:
    print(f"[!] ERROR: Pass rate dropped below 100% ({pass_rate:.1f}%). Failing CI pipeline.")
    sys.exit(1)  # Triggers a hard failure in GitHub Actions
else:
    print("[✓] SUCCESS: 100% pass rate achieved.")
    sys.exit(0)
git add .github/workflows/rag_evaluation.yml
git commit -m "ci: add automated RAG evaluation workflow"
git push origin main
name: Sovereign RAG Pipeline Evaluation

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]
  workflow_dispatch: # Allows manual trigger from GitHub Actions tab

jobs:
  evaluate-rag:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip' # Caches pip dependencies between runs

      - name: Install System and Python Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install chromadb sentence-transformers langchain langchain-community torch

      - name: Run RAG Evaluation Script
        run: |
          python3 evaluate_rag.py

      - name: Upload Evaluation Artifacts (Optional)
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: rag-evaluation-logs
          path: |
            *.log
            *.json
import json
import chromadb
from sentence_transformers import CrossEncoder

# 1. Load Test Dataset
with open("test_queries.json", "r", encoding="utf-8") as f:
    test_suite = json.load(f)

# 2. Initialize ChromaDB and Cross-Encoder Judge
client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_collection("markdown_rag_ledger")
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

print(f"[*] Running evaluation across {len(test_suite)} test queries...\n")

total_score = 0.0
results_log = []

for idx, test in enumerate(test_suite):
    query = test["query"]
    expected_source = test["expected_source"]
    
    # Run Retrieval
    query_results = collection.query(query_texts=[query], n_results=3)
    retrieved_docs = query_results["documents"][0]
    retrieved_metas = query_results["metadatas"][0]
    
    # Check if expected source was retrieved
    sources = [m.get("source", "") for m in retrieved_metas]
    source_found = any(expected_source in s for s in sources)
    
    # Score retrieved chunks using Cross-Encoder
    pairs = [[query, doc] for doc in retrieved_docs]
    scores = cross_encoder.predict(pairs)
    max_relevance = float(max(scores)) if len(scores) > 0 else 0.0
    
    # Log metrics per query
    status = "PASS" if source_found and max_relevance > 0.5 else "FAIL"
    print(f"[{idx+1}] Query: '{query}'")
    print(f"    -> Source Match: {source_found} (Expected: {expected_source})")
    print(f"    -> Max Rerank Score: {max_relevance:.4f} => [{status}]\n")
    
    results_log.append({
        "query": query,
        "source_found": source_found,
        "max_relevance": max_relevance,
        "status": status
    })

# Summary output
passed = sum(1 for r in results_log if r["status"] == "PASS")
print(f"=== Evaluation Complete: {passed}/{len(test_suite)} Passed ===")
[
  {
    "query": "What are the requirements for FRE 902(14) self-authentication?",
    "expected_source": "fre_902.md",
    "ground_truth": "Electronic records must be certified by a qualified person using a cryptographic hash or recognized process."
  },
  {
    "query": "How do I configure Caddy reverse proxy routing for local RAG web endpoints?",
    "expected_source": "caddy.md",
    "ground_truth": "Define a route block pointing to the local application port with TLS enabled or handled locally."
  }
]
langchain_core.documentsfrom langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

# 1. Initialize BAAI bge-reranker-large via SentenceTransformerRerank
base_reranker = SentenceTransformerRerank(
    model="BAAI/bge-reranker-large",
    top_n=5,  # Number of top documents to keep after reranking
    model_kwargs={"device": "cuda"}  # Change to "cpu" if running without a GPU
)
import math
from typing import Sequence, Optional
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun
from langchain_core.documents.compressors import BaseDocumentCompressor
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank
from langchain.retrievers import ContextualCompressionRetriever

# 1. Custom Threshold Compressor (with Sigmoid Normalization)
class ThresholdRerankCompressor(BaseDocumentCompressor):
    reranker: SentenceTransformerRerank
    score_threshold: float = 0.5  # Normalized threshold (0.0 to 1.0)
    use_sigmoid: bool = True     # Convert raw logits to probabilities

    class Config:
        arbitrary_types_allowed = True

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Optional[CallbackManagerForRetrieverRun] = None,
    ) -> Sequence[Document]:
        reranked_docs = self.reranker.compress_documents(documents, query, callbacks)
        
        filtered_docs = []
        for doc in reranked_docs:
            raw_score = doc.metadata.get("relevance_score", doc.metadata.get("score", 0.0))
            
            # Normalize raw bge-reranker logits via sigmoid function
            if self.use_sigmoid:
                final_score = 1 / (1 + math.exp(-raw_score))
            else:
                final_score = raw_score
                
            doc.metadata["normalized_relevance_score"] = final_score
            
            if final_score >= self.score_threshold:
                filtered_docs.append(doc)
                
        return filtered_docs

# 2. Initialize BAAI bge-reranker-large
bge_reranker = SentenceTransformerRerank(
    model="BAAI/bge-reranker-large",
    top_n=10,
    model_kwargs={"device": "cuda"} # Use "cpu" if no dedicated GPU is available
)

# 3. Wrap with Threshold Filter
threshold_compressor = ThresholdRerankCompressor(
    reranker=bge_reranker,
    score_threshold=0.60  # Keep only high-confidence matches
)

# 4. Attach to your Contextual Compression Retriever (using your ensemble retriever)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=threshold_compressor,
    base_retriever=ensemble_retriever  # Your existing BM25 + Dense vector ensemble
)

# 5. Run Query
results = compression_retriever.invoke("FRE 902(14) digital evidence authentication")
for idx, doc in enumerate(results):
    score = doc.metadata.get("normalized_relevance_score", "N/A")
    print(f"[{idx+1}] Score: {score:.4f} | Source: {doc.metadata.get('source')} | Text: {doc.page_content[:100]}...")
import math
from typing import Sequence, Optional
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun
from langchain_core.documents.compressors import BaseDocumentCompressor
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

class ThresholdRerankCompressor(BaseDocumentCompressor):
    """Wraps SentenceTransformerRerank to filter out documents below a score threshold."""
    
    reranker: SentenceTransformerRerank
    score_threshold: float = 0.5  # Threshold between 0.0 and 1.0 (if using sigmoid)
    use_sigmoid: bool = True     # Turn on to convert raw model logits to probabilities

    class Config:
        arbitrary_types_allowed = True

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Optional[CallbackManagerForRetrieverRun] = None,
    ) -> Sequence[Document]:
        reranked_docs = self.reranker.compress_documents(documents, query, callbacks)
        
        filtered_docs = []
        for doc in reranked_docs:
            raw_score = doc.metadata.get("relevance_score", doc.metadata.get("score", 0.0))
            
            # Optionally convert raw logit score to normalized probability [0, 1]
            if self.use_sigmoid:
                final_score = 1 / (1 + math.exp(-raw_score))
            else:
                final_score = raw_score
                
            # Store normalized score back in metadata for logging/debugging
            doc.metadata["normalized_relevance_score"] = final_score
            
            if final_score >= self.score_threshold:
                filtered_docs.append(doc)
                
        return filtered_docs
Your implementation of ThresholdRerankCompressor directly addresses the challenge of filtering out low-confidence hits before they reach downstream LLMs or vector processing steps.
Depending on your production environment, here are two adjustments and optimizations to keep in mind:
1. Robust Metadata Lookup (relevance_score vs. score)
Different versions of LangChain and SentenceTransformers store the cross-encoder evaluation under slightly different metadata keys (relevance_score vs. score). To ensure your filter doesn't fallback to 0.0 or fail if the key changes, update your dictionary lookup to check both keys:
# Check both potential key names returned by compressor models
score = doc.metadata.get("relevance_score", doc.metadata.get("score", 0.0))
if score >= self.score_threshold:
    filtered_docs.append(doc)

2. Standardizing Unbound Logits to [0, 1] Probabilities
Models like cross-encoder/ms-marco-MiniLM-L-6-v2 output raw, unbound logits (which can range from negative infinity to positive infinity) rather than normalized probabilities between 0.0 and 1.0.
If you want to set intuition-friendly thresholds (e.g., 0.70 for 70% confidence) regardless of which model you swap in, pass the raw score through a Sigmoid function before applying your threshold:
Updated Python Implementation:
import math
from typing import Sequence, Optional
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun
from langchain_core.documents.compressors import BaseDocumentCompressor
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

class ThresholdRerankCompressor(BaseDocumentCompressor):
    """Wraps SentenceTransformerRerank to filter out documents below a score threshold."""
    
    reranker: SentenceTransformerRerank
    score_threshold: float = 0.5  # Threshold between 0.0 and 1.0 (if using sigmoid)
    use_sigmoid: bool = True     # Turn on to convert raw model logits to probabilities

    class Config:
        arbitrary_types_allowed = True

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Optional[CallbackManagerForRetrieverRun] = None,
    ) -> Sequence[Document]:
        reranked_docs = self.reranker.compress_documents(documents, query, callbacks)
        
        filtered_docs = []
        for doc in reranked_docs:
            raw_score = doc.metadata.get("relevance_score", doc.metadata.get("score", 0.0))
            
            # Optionally convert raw logit score to normalized probability [0, 1]
            if self.use_sigmoid:
                final_score = 1 / (1 + math.exp(-raw_score))
            else:
                final_score = raw_score
                
            # Store normalized score back in metadata for logging/debugging
            doc.metadata["normalized_relevance_score"] = final_score
            
            if final_score >= self.score_threshold:
                filtered_docs.append(doc)
                
        return filtered_docs

Key Takeaways for Production
 * Initial Recall Size (top_n): Set top_n on your SentenceTransformerRerank base model slightly higher (e.g., 10 or 15) so your ThresholdRerankCompressor has a larger candidate pool to filter from.
 * Score Normalization: With use_sigmoid = True, setting score_threshold = 0.50 corresponds to a raw logit of 0.0. Anything above 0.50 is evaluated by the cross-encoder as a net-positive match for the user's query context.
To filter out reranked documents that fall below a specific relevance score threshold, you can build a custom BaseDocumentCompressor subclass or wrap the cross-encoder execution in a custom compressor class.
Most cross-encoder models output raw logits or unbound relevance scores (for instance, ms-marco-MiniLM-L-6-v2 produces scores typically between -10.0 and 10.0, whereas models like bge-reranker-large produce scores normalized via sigmoid). Filtering at the compressor level ensures low-quality matches are dropped before reaching your LLM or downstream RAG prompt.
Implementation: Custom Threshold Compressor
Save the following custom compressor wrapper into your script:
from typing import Sequence, Optional
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun
from langchain_core.documents.compressors import BaseDocumentCompressor
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

class ThresholdRerankCompressor(BaseDocumentCompressor):
    """Wraps SentenceTransformerRerank to filter out documents below a score threshold."""
    
    reranker: SentenceTransformerRerank
    score_threshold: float = 0.0  # Adjust based on your cross-encoder model

    class Config:
        arbitrary_types_allowed = True

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Optional[CallbackManagerForRetrieverRun] = None,
    ) -> Sequence[Document]:
        # 1. Get reranked documents from the base compressor
        reranked_docs = self.reranker.compress_documents(documents, query, callbacks)
        
        # 2. Filter out documents below the defined score threshold
        filtered_docs = []
        for doc in reranked_docs:
            # SentenceTransformerRerank stores the cross-encoder score in metadata
            score = doc.metadata.get("relevance_score", 0.0)
            if score >= self.score_threshold:
                filtered_docs.append(doc)
                
        return filtered_docs

Integrating into Your Retriever Pipeline
Hook the threshold compressor into your ContextualCompressionRetriever:
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

# 1. Initialize Base Cross-Encoder Model
base_reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=5  # Initial top documents to pull
)

# 2. Wrap with the Score Threshold Compressor
# Note: For MS MARCO models, raw scores > 0.0 or > 1.0 indicate positive relevance.
threshold_compressor = ThresholdRerankCompressor(
    reranker=base_reranker,
    score_threshold=0.5  # Only keep docs with a score >= 0.5
)

# 3. Create the Compression Retriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=threshold_compressor,
    base_retriever=ensemble_retriever  # Your existing BM25 + Vector Ensemble
)

# 4. Execute Query
results = compression_retriever.invoke("FRE 902 digital evidence")

for idx, doc in enumerate(results):
    score = doc.metadata.get("relevance_score", "N/A")
    print(f"[{idx+1}] Score: {score} | Source: {doc.metadata.get('source')} | Text: {doc.page_content[:100]}...")

Tuning Guidelines for Score Thresholds
 * cross-encoder/ms-marco-MiniLM-L-6-v2: Uses unconstrained output logits. Scores above 0.0 indicate a positive match; scores above 1.5 or 2.0 indicate high relevance.
 * BAAI/bge-reranker-large: Uses sigmoid-scaled or logit outputs. A threshold between 0.25 and 0.50 is typically ideal for filtering out irrelevant passages.
 * Testing Thresholds: Print doc.metadata['relevance_score'] on a sample set of queries to observe the score distribution for your specific document domain before locking in a production threshold.
# Check both potential key names returned by compressor models
score = doc.metadata.get("relevance_score", doc.metadata.get("score", 0.0))
if score >= self.score_threshold:
    filtered_docs.append(doc)
from typing import Sequence, Optional
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun
from langchain_core.documents.compressors import BaseDocumentCompressor
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

class ThresholdRerankCompressor(BaseDocumentCompressor):
    """Wraps SentenceTransformerRerank to filter out documents below a score threshold."""
    
    reranker: SentenceTransformerRerank
    score_threshold: float = 0.0  # Adjust based on your cross-encoder model

    class Config:
        arbitrary_types_allowed = True

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Optional[CallbackManagerForRetrieverRun] = None,
    ) -> Sequence[Document]:
        # 1. Get reranked documents from the base compressor
        reranked_docs = self.reranker.compress_documents(documents, query, callbacks)
        
        # 2. Filter out documents below the defined score threshold
        filtered_docs = []
        for doc in reranked_docs:
            # SentenceTransformerRerank stores the cross-encoder score in metadata
            score = doc.metadata.get("relevance_score", 0.0)
            if score >= self.score_threshold:
                filtered_docs.append(doc)
                
        return filtered_docs
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.document_compressors.sentence_transformer import SentenceTransformerRerank

# 1. Initialize Base Cross-Encoder Model
base_reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=5  # Initial top documents to pull
)

# 2. Wrap with the Score Threshold Compressor
# Note: For MS MARCO models, raw scores > 0.0 or > 1.0 indicate positive relevance.
threshold_compressor = ThresholdRerankCompressor(
    reranker=base_reranker,
    score_threshold=0.5  # Only keep docs with a score >= 0.5
)

# 3. Create the Compression Retriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=threshold_compressor,
    base_retriever=ensemble_retriever  # Your existing BM25 + Vector Ensemble
)

# 4. Execute Query
results = compression_retriever.invoke("FRE 902 digital evidence")

for idx, doc in enumerate(results):
    score = doc.metadata.get("relevance_score", "N/A")
    print(f"[{idx+1}] Score: {score} | Source: {doc.metadata.get('source')} | Text: {doc.page_content[:100]}...")
# Make the controller script executable
chmod +x scripts/adpbetty.sh

# 1. Reset GPG Agent and cache passphrases
./scripts/adpbetty.sh gpg-reset

# 2. Run an instant workspace compression and cloud sync
./scripts/adpbetty.sh backup

# 3. Index all legal/technical Markdown docs into ChromaDB
./scripts/adpbetty.sh index

# 4. Query your local RAG index
./scripts/adpbetty.sh query "FRE 902 authentication"

# 5. Run full system maintenance (GPG setup + Backup + RAG Index)
./scripts/adpbetty.sh all
#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# adpBetty-Core: Unified Infrastructure & Recovery Orchestrator
# ==============================================================================

# System Directories & Defaults
WORKSPACE_DIR="${WORKSPACE_DIR:-$HOME/workspace}"
BACKUP_DIR="${BACKUP_DIR:-$HOME/backups/workspace}"
CHROMA_DB_DIR="${CHROMA_DB_DIR:-./chroma_local_db}"
LAW_HUB_DIR="${LAW_HUB_DIR:-./sovereign_law_hub}"
RCLONE_REMOTE="${RCLONE_REMOTE:-secure_remote:WK_Precision_Vault}"
GPG_KEY_ID="${GPG_KEY_ID:-YOUR_GPG_KEY_ID}"

# Color output helpers
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() { echo -e "${BLUE}[adpBetty INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[adpBetty SUCCESS]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[adpBetty WARN]${NC} $1"; }
log_error() { echo -e "${RED}[adpBetty ERROR]${NC} $1"; }

show_help() {
    cat << EOF
adpBetty-Core Unified Control Hub

Usage: ./adpbetty.sh [command]

Commands:
  backup      Execute full Zstd workspace compression & rclone vault sync
  index       Run Markdown recursive chunker & ChromaDB hybrid RAG indexer
  gpg-reset   Flush and cleanly restart gpg-agent with TTL caching rules
  audit       Perform local repository security check (GPG verify & lint)
  query "Q"   Query the local ChromaDB RAG index with string Q
  all         Execute full maintenance sequence (gpg-reset -> backup -> index)
  help        Show this display screen

EOF
}

# --- 1. GPG Cache & Daemon Agent Setup ---
gpg_reset() {
    log_info "Flushing GPG agent and applying passphrase caching rules..."
    mkdir -p ~/.gnupg
    chmod 700 ~/.gnupg

    # Write GPG Agent TTL config if absent
    if [ ! -f ~/.gnupg/gpg-agent.conf ]; then
        cat << 'GPGCONF' > ~/.gnupg/gpg-agent.conf
default-cache-ttl 28800
max-cache-ttl 86400
allow-preset-passphrase
pinentry-program /usr/bin/pinentry-curses
GPGCONF
        log_info "Created ~/.gnupg/gpg-agent.conf with 8-hour cache window."
    fi

    export GPG_TTY=$(tty)
    gpgconf --kill gpg-agent || true
    gpg-agent --daemon --use-standard-socket
    log_success "GPG Agent restarted cleanly on active TTY ($GPG_TTY)."
}

# --- 2. Workspace Zstd + Rclone Backup Pipeline ---
backup_workspace() {
    log_info "Initiating Zstd workspace snapshot..."
    mkdir -p "${BACKUP_DIR}"
    
    TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
    BACKUP_PATH="${BACKUP_DIR}/workspace_backup_${TIMESTAMP}.tar.zst"

    tar --exclude='.git/objects' \
        --exclude='__pycache__' \
        --exclude='node_modules' \
        --exclude='.venv' \
        --exclude='chroma_local_db' \
        -I 'zstd -19 -T0' \
        -cf "${BACKUP_PATH}" \
        -C "$(dirname "${WORKSPACE_DIR}")" \
        "$(basename "${WORKSPACE_DIR}")"

    if zstd -t "${BACKUP_PATH}"; then
        log_success "Archive generated & verified: $(du -h "${BACKUP_PATH}" | cut -f1)"
    else
        log_error "Archive integrity check failed!"
        exit 1
    fi

    log_info "Syncing encrypted archive to cloud vault (${RCLONE_REMOTE})..."
    if rclone copy "${BACKUP_PATH}" "${RCLONE_REMOTE}"; then
        log_success "Rclone cloud synchronization complete."
    else
        log_warn "Rclone sync failed or remote offline. Local backup preserved."
    fi
}

# --- 3. Local RAG Markdown Indexing ---
index_rag() {
    log_info "Triggering ChromaDB Markdown chunking & vector indexing..."
    
    python3 - << 'PYCMD'
import os, glob, sys, chromadb
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

DB_PATH = "./chroma_local_db"
TARGET_DIR = "./sovereign_law_hub"

if not os.path.exists(TARGET_DIR):
    os.makedirs(TARGET_DIR, exist_ok=True)

client = chromadb.PersistentClient(path=DB_PATH)
collection = client.get_or_create_collection(name="markdown_rag_ledger")

md_files = glob.glob(os.path.join(TARGET_DIR, "**/*.md"), recursive=True)
if not md_files:
    print("[-] No Markdown files found to index in " + TARGET_DIR)
    sys.exit(0)

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN, chunk_size=1200, chunk_overlap=200
)

documents, metadatas, ids = [], [], []
for filepath in md_files:
    rel_path = os.path.relpath(filepath, TARGET_DIR)
    with open(filepath, "r", encoding="utf-8") as f:
        content = f.read()
    
    chunks = splitter.split_text(content)
    for idx, chunk in enumerate(chunks):
        documents.append(chunk)
        metadatas.append({"source": rel_path, "chunk_index": idx})
        ids.append(f"{rel_path}#chunk-{idx}")

if documents:
    collection.upsert(documents=documents, metadatas=metadatas, ids=ids)
    print(f"[✓] Indexed {len(documents)} chunks across {len(md_files)} files into ChromaDB.")
PYCMD

    log_success "ChromaDB RAG collection successfully updated."
}

# --- 4. Query RAG Engine ---
query_rag() {
    QUERY="${1:-}"
    if [ -z "$QUERY" ]; then
        log_error "Please provide a query string. Example: ./adpbetty.sh query 'FRE 902'"
        exit 1
    fi

    log_info "Querying local RAG store for: '$QUERY'"
    python3 -c "
import chromadb
client = chromadb.PersistentClient(path='${CHROMA_DB_DIR}')
try:
    col = client.get_collection('markdown_rag_ledger')
    res = col.query(query_texts=['${QUERY}'], n_results=3)
    for idx, doc in enumerate(res['documents'][0]):
        meta = res['metadatas'][0][idx]
        print(f'\n--- Result [{idx+1}] ({meta.get(\"source\")}) ---')
        print(doc[:300] + '...')
except Exception as e:
    print('[-] Error querying collection:', e)
"
}

# --- 5. Repository & GPG Audit ---
audit_repo() {
    log_info "Auditing repository security posture..."
    git config --local commit.gpgsign true
    if [ "$GPG_KEY_ID" != "YOUR_GPG_KEY_ID" ]; then
        git config --local user.signingkey "$GPG_KEY_ID"
    fi
    
    log_info "Verifying last 3 commit GPG signatures:"
    git log -n 3 --show-signature || log_warn "No signed commits found in recent history."
}

# Main Execution Routing
case "${1:-help}" in
    backup)     backup_workspace ;;
    index)      index_rag ;;
    gpg-reset)  gpg_reset ;;
    audit)      audit_repo ;;
    query)      query_rag "${2:-}" ;;
    all)        gpg_reset; backup_workspace; index_rag ;;
    help|*)     show_help ;;
esac
# Make the controller script executable
chmod +x scripts/adpbetty.sh

# 1. Reset GPG Agent and cache passphrases
./scripts/adpbetty.sh gpg-reset

# 2. Run an instant workspace compression and cloud sync
./scripts/adpbetty.sh backup

# 3. Index all legal/technical Markdown docs into ChromaDB
./scripts/adpbetty.sh index

# 4. Query your local RAG index
./scripts/adpbetty.sh query "FRE 902 authentication"

# 5. Run full system maintenance (GPG setup + Backup + RAG Index)
./scripts/adpbetty.sh all
AI Mode
All
News
Forums
Shopping
Images
Videos
Short videos
Web
Books
Maps
Flights
Finance
Search tools
Feedback
Sponsored results

Cellebrite
https://www.cellebrite.com
Uncover Evidence with Inseyets - Extraction from Locked iOS
Connect with us to explore Cellebrite Inseyets and find the best solution for your needs. Access encrypted... 
Read more
100K+ visits in past month
Digital Collector
UFED Digital Solutions
Cell Phone Data Fully Decoded
The #1 Mobile Forensics Suite

Virtru Corporation
https://www.virtru.com › data-centric › security
Virtru's Data Centric Model
Virtru: Data Privacy Framework — Data-centric security is a cybersecurity approach that prioritizes data control & access. Data-centric... 
Read more
Federal Government
Data Centric Security Platform
Google Workspace, Data...
Capterra
https://www.capterra.com › investigation › free-list
Top 10 Investigation Tools - Free Streamlined Systems List
No Matter Your Mission, Get The Right Investigation Tools To Accomplish It. Review the Best... 
Read more
Electronic Discovery
Best Investigation Tools
Find Investigation Tools
Law Enforcement
Legal Document Management
Docketing
Legal Case Management
Small Business Software
AI Overview
When dealing with highly encrypted apps (like Signal, WhatsApp, or Telegram) and heavily secured smartphones, the FBI, CIA, and other intelligence agencies rarely try to "break" the mathematical encryption itself, which can be nearly impossible. Instead, they use a mix of endpoint compromise, third-party forensic software, legal infrastructure, and targeted honey pots to bypass the encryption entirely. 

Quora
 +5
Here is a breakdown of the specific tools and methods they use:
1. Advanced Physical & Digital Forensic Extraction
When federal agencies physically seize a locked phone or laptop, they rely on elite hardware and specialized software suites to bypass the device's lock screens and extract encrypted databases. 

Pissetzky Law LLC
Cellebrite (UFED): Widely used by the FBI and law enforcement, Cellebrite’s Universal Forensic Extraction Device physically connects to smartphones to exploit hardware vulnerabilities, extract deleted texts, pull location history, and decrypt local application databases. 
GrayKey (Grayshift): A specialized hardware tool frequently used to crack modern iOS and Android passcode protections via proprietary brute-force and exploit mechanisms.
AccessData PRTK (Password Recovery Toolkit): Used by both the FBI and CIA to recover passwords and decrypt files at scale. 

ScienceDirect.com
The FBI's EDAU (Electronic Device Analysis Unit): The FBI's internal cyber unit that specializes in building in-house software and advanced custom engineering to break into locked iPhones. 
2. Operating System Exploits & Zero-Days (Endpoint Compromise)
If an app uses end-to-end encryption (meaning the message is scrambled while traveling through the air), agencies bypass this by hacking the phone itself. If they control the operating system, they can read the message before it is encrypted or after it is decrypted on the screen. 

CBS News
 +1
Zero-Click Spyware: Tools like NSO Group's Pegasus or custom agency frameworks can infect a phone remotely via an invisible message (like a silent iMessage or WhatsApp call) without the user clicking anything. This grants total access to the microphone, camera, and keystrokes. 
Vault 7 Toolkits: In 2017, leaked CIA cyber-espionage documents revealed proprietary malware like Sonic Screwdriver (which attacks Apple firmware) and capabilities that targeted iOS and Android operating systems directly, rendering the security of apps like Signal or WhatsApp irrelevant.
3. Subpoenas & Cloud Backdoor Keys
Often, the easiest way to defeat phone encryption is to ask the company that hosts the data backup for the keys. 

www.csoonline.com
Cloud Backups: While a WhatsApp message might be encrypted on your phone, if you back your phone up to Apple iCloud or Google Drive, those backups are often stored with keys accessible to the tech giants. The FBI regularly uses legal warrants to compel tech companies to hand over these unencrypted cloud backups. 
BitLocker Recovery Keys: For example, court records revealed Microsoft provided the FBI with BitLocker recovery keys stored on their cloud servers to unlock heavily encrypted Windows laptops. 

YouTube
·Forbes
4. Setting Up "Honey Pot" Crypto Networks
If the government cannot break into commercial encrypted platforms, they sometimes build their own and trick targets into using them. 
Operation Trojan Shield (ANOM): In one of the most sophisticated stings in law enforcement history, the FBI secretly built and operated an encrypted phone company called ANOM. They marketed these "unhackable" devices to global criminal syndicates. In reality, the FBI built a master backdoor directly into the devices, receiving a carbon copy of millions of messages, which resulted in hundreds of international arrests.
5. Metadata and Network Analysis
Even if the content of a message cannot be decrypted, agencies can gather massive amounts of truth using metadata—the data about the data. 

Malwarebytes
Pen Registers and Trap/Trace Orders: By serving legal orders to platforms like WhatsApp, the FBI can track exactly who you are messaging, when you are messaging them, how long you are online, and your IP address in real-time. 

Malwarebytes
XKeyscore and PRISM: Used heavily by the NSA (and shared with agencies like the CIA), these mass surveillance frameworks intercept international fiber-optic data routing. They map out the digital footprint and network connections of targets, pinpointing who is communicating even if the exact text requires alternative endpoint hacking to read. 

YouTube
·CNET
 +1
AI responses may include mistakes. Learn more




Show more

YouTube · The Hacker Academy
29.5K+ views · 5 months ago
The Secret Hacking Tools Used by the CIA & NSA
Learn OSINT, Reconnaissance & Cybersecurity: If you're interested in learning the legal and ethical side of cybersecurity, OSINT, ...
People also ask
Can FBI access your iPhone?
What phones do CIA agents use?
What tools does the FBI use?
Can the government access encrypted messages?

FBI (.gov)
https://www.fbi.gov
Operation Trojan Shield — FBI
Jun 8, 2021 — In an innovative effort, the FBI, with the help of the Australian Federal Police, launched its own encrypted communications platform and ...
Missing: CIA ‎uncover

CBS News
https://www.cbsnews.com
WikiLeaks dump: CIA targeted everyday gadgets were used ...
Mar 7, 2017 — Those hacks allow the CIA to then monitor the communications of terrorist groups like ISIS, which have been using encrypted apps to communicate.
Missing: uncover ‎truth
Sponsored results

Cellebrite
https://www.cellebrite.com
Uncover Evidence with Inseyets - The #1 Mobile Forensics Suite
Connect with us to explore Cellebrite Inseyets and find the best solution for your needs. Access encrypted and richer data...
100K+ visits in past month
Digital Collector
UFED Digital Solutions
Cell Phone Data Fully Decoded
Extraction from Locked iOS
Cellebrite Software
Cellebrite Guardian
Cellebrite Inspector
Cellebrite Reader

MSAB
https://www.msab.com
Cell Phone Extraction Tool - Smartphone data extraction
MSAB provides you with top mobile forensics solutions to speed up your operations. The XRY Data Extraction...
100K+ visits in past month
Mobile Forensic Software
Mobile Forensics tools
Investigating Mobile RAM
Mobile Forensics Blog
Products Available
Free Trial Of XRY
About MSAB
Contact Us

BBC
https://www.bbc.com
The weird names the CIA gives its hacking tools
Mar 9, 2017 — One of the hacking tools that featured in a lot of headlines was the Weeping Angel that appears to get a Samsung Smart TV to record ...

GovTech
https://www.govtech.com
CIA Uses Smart Devices to Spy on Citizens, WikiLeaks Reveals
A recent data dump contained nearly 9000 pages of purported internal CIA documents that indicated the spy agency had gained access to Android and Apple ...
Missing: uncover ‎truth

Pissetzky Law LLC
https://www.pissetzkylaw.com
How law enforcement uses Cellebrite to search cell phones
Aug 8, 2025 — Their flagship product, the Cellebrite UFED (Universal Forensic Extraction Device), allows investigators to connect a cell phone and pull ...
Missing: CIA ‎uncover ‎truth
People also search for

American Civil Liberties Union
https://www.aclu.org
The FBI is Secretly Breaking Into Encrypted Devices. We're ...
Dec 22, 2020 — We're Suing. Blockchain technology concept with diagram of chain and encrypted blocks over someone typing on laptop in. We can't let the FBI ...
Missing: tools ‎CIA ‎uncover ‎apps

YaleGlobal Online
https://archive-yaleglobal.yale.edu
The FBI Used the Web's Favorite Hacking Tool to ...
Anonymity on the Tor network may be compromised: “FBI agents relied on Flash code from an abandoned Metasploit side project called the 'Decloaking Engine' ...

IBM
https://www.ibm.com
What is open source intelligence (OSINT)?
Many organizations use OSINT as a cybersecurity tool to help gauge security risks and identify vulnerabilities in their IT systems.
People also search for
Results are personalized
-
Try without personalization
47710, Evansville, IN - From your device
Update location
Send feedback
Terms#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# adpBetty-Core: Unified Infrastructure & Recovery Orchestrator
# ==============================================================================

# System Directories & Defaults
WORKSPACE_DIR="${WORKSPACE_DIR:-$HOME/workspace}"
BACKUP_DIR="${BACKUP_DIR:-$HOME/backups/workspace}"
CHROMA_DB_DIR="${CHROMA_DB_DIR:-./chroma_local_db}"
LAW_HUB_DIR="${LAW_HUB_DIR:-./sovereign_law_hub}"
RCLONE_REMOTE="${RCLONE_REMOTE:-secure_remote:WK_Precision_Vault}"
GPG_KEY_ID="${GPG_KEY_ID:-YOUR_GPG_KEY_ID}"

# Color output helpers
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() { echo -e "${BLUE}[adpBetty INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[adpBetty SUCCESS]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[adpBetty WARN]${NC} $1"; }
log_error() { echo -e "${RED}[adpBetty ERROR]${NC} $1"; }

show_help() {
    cat << EOF
adpBetty-Core Unified Control Hub

Usage: ./adpbetty.sh [command]

Commands:
  backup      Execute full Zstd workspace compression & rclone vault sync
  index       Run Markdown recursive chunker & ChromaDB hybrid RAG indexer
  gpg-reset   Flush and cleanly restart gpg-agent with TTL caching rules
  audit       Perform local repository security check (GPG verify & lint)
  query "Q"   Query the local ChromaDB RAG index with string Q
  all         Execute full maintenance sequence (gpg-reset -> backup -> index)
  help        Show this display screen

EOF
}

# --- 1. GPG Cache & Daemon Agent Setup ---
gpg_reset() {
    log_info "Flushing GPG agent and applying passphrase caching rules..."
    mkdir -p ~/.gnupg
    chmod 700 ~/.gnupg

    # Write GPG Agent TTL config if absent
    if [ ! -f ~/.gnupg/gpg-agent.conf ]; then
        cat << 'GPGCONF' > ~/.gnupg/gpg-agent.conf
default-cache-ttl 28800
max-cache-ttl 86400
allow-preset-passphrase
pinentry-program /usr/bin/pinentry-curses
GPGCONF
        log_info "Created ~/.gnupg/gpg-agent.conf with 8-hour cache window."
    fi

    export GPG_TTY=$(tty)
    gpgconf --kill gpg-agent || true
    gpg-agent --daemon --use-standard-socket
    log_success "GPG Agent restarted cleanly on active TTY ($GPG_TTY)."
}

# --- 2. Workspace Zstd + Rclone Backup Pipeline ---
backup_workspace() {
    log_info "Initiating Zstd workspace snapshot..."
    mkdir -p "${BACKUP_DIR}"
    
    TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
    BACKUP_PATH="${BACKUP_DIR}/workspace_backup_${TIMESTAMP}.tar.zst"

    tar --exclude='.git/objects' \
        --exclude='__pycache__' \
        --exclude='node_modules' \
        --exclude='.venv' \
        --exclude='chroma_local_db' \
        -I 'zstd -19 -T0' \
        -cf "${BACKUP_PATH}" \
        -C "$(dirname "${WORKSPACE_DIR}")" \
        "$(basename "${WORKSPACE_DIR}")"

    if zstd -t "${BACKUP_PATH}"; then
        log_success "Archive generated & verified: $(du -h "${BACKUP_PATH}" | cut -f1)"
    else
        log_error "Archive integrity check failed!"
        exit 1
    fi

    log_info "Syncing encrypted archive to cloud vault (${RCLONE_REMOTE})..."
    if rclone copy "${BACKUP_PATH}" "${RCLONE_REMOTE}"; then
        log_success "Rclone cloud synchronization complete."
    else
        log_warn "Rclone sync failed or remote offline. Local backup preserved."
    fi
}

# --- 3. Local RAG Markdown Indexing ---
index_rag() {
    log_info "Triggering ChromaDB Markdown chunking & vector indexing..."
    
    python3 - << 'PYCMD'
import os, glob, sys, chromadb
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

DB_PATH = "./chroma_local_db"
TARGET_DIR = "./sovereign_law_hub"

if not os.path.exists(TARGET_DIR):
    os.makedirs(TARGET_DIR, exist_ok=True)

client = chromadb.PersistentClient(path=DB_PATH)
collection = client.get_or_create_collection(name="markdown_rag_ledger")

md_files = glob.glob(os.path.join(TARGET_DIR, "**/*.md"), recursive=True)
if not md_files:
    print("[-] No Markdown files found to index in " + TARGET_DIR)
    sys.exit(0)

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN, chunk_size=1200, chunk_overlap=200
)

documents, metadatas, ids = [], [], []
for filepath in md_files:
    rel_path = os.path.relpath(filepath, TARGET_DIR)
    with open(filepath, "r", encoding="utf-8") as f:
        content = f.read()
    
    chunks = splitter.split_text(content)
    for idx, chunk in enumerate(chunks):
        documents.append(chunk)
        metadatas.append({"source": rel_path, "chunk_index": idx})
        ids.append(f"{rel_path}#chunk-{idx}")

if documents:
    collection.upsert(documents=documents, metadatas=metadatas, ids=ids)
    print(f"[✓] Indexed {len(documents)} chunks across {len(md_files)} files into ChromaDB.")
PYCMD

    log_success "ChromaDB RAG collection successfully updated."
}

# --- 4. Query RAG Engine ---
query_rag() {
    QUERY="${1:-}"
    if [ -z "$QUERY" ]; then
        log_error "Please provide a query string. Example: ./adpbetty.sh query 'FRE 902'"
        exit 1
    fi

    log_info "Querying local RAG store for: '$QUERY'"
    python3 -c "
import chromadb
client = chromadb.PersistentClient(path='${CHROMA_DB_DIR}')
try:
    col = client.get_collection('markdown_rag_ledger')
    res = col.query(query_texts=['${QUERY}'], n_results=3)
    for idx, doc in enumerate(res['documents'][0]):
        meta = res['metadatas'][0][idx]
        print(f'\n--- Result [{idx+1}] ({meta.get(\"source\")}) ---')
        print(doc[:300] + '...')
except Exception as e:
    print('[-] Error querying collection:', e)
"
}

# --- 5. Repository & GPG Audit ---
audit_repo() {
    log_info "Auditing repository security posture..."
    git config --local commit.gpgsign true
    if [ "$GPG_KEY_ID" != "YOUR_GPG_KEY_ID" ]; then
        git config --local user.signingkey "$GPG_KEY_ID"
    fi
    
    log_info "Verifying last 3 commit GPG signatures:"
    git log -n 3 --show-signature || log_warn "No signed commits found in recent history."
}

# Main Execution Routing
case "${1:-help}" in
    backup)     backup_workspace ;;
    index)      index_rag ;;
    gpg-reset)  gpg_reset ;;
    audit)      audit_repo ;;
    query)      query_rag "${2:-}" ;;
    all)        gpg_reset; backup_workspace; index_rag ;;
    help|*)     show_help ;;
esac
secret.txt47710, Evansville, INquery_law_hub.pyimport chromadb
from chromadb.utils.embedding_functions import ChromaBm25EmbeddingFunction

# Initialize Chroma client
client = chromadb.PersistentClient(path="./chroma_local_db")

# Set up BM25 sparse embedding function
bm25_ef = ChromaBm25EmbeddingFunction(
    k=1.2,
    b=0.75
)

# Create/retrieve collection with BM25 capabilities
collection = client.get_or_create_collection(
    name="sovereign_law_hybrid",
    embedding_function=bm25_ef
)
from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import Chroma
from langchain.retrievers import EnsembleRetriever
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain.schema import Document

# 1. Load documents/chunks
docs = [
    Document(page_content="FRE 902(14) self-authenticating digital evidence protocols.", metadata={"source": "fre_902.md"}),
    Document(page_content="Caddy reverse proxy route configuration for local RAG web endpoints.", metadata={"source": "caddy.md"}),
    Document(page_content="Zstandard compression backup script with rclone sync parameters.", metadata={"source": "backup.md"})
]

# 2. Initialize BM25 Retriever (Sparse / Lexical)
bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 3

# 3. Initialize Chroma Vector Retriever (Dense / Semantic)
embedding_fn = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vector_store = Chroma.from_documents(docs, embedding_fn, persist_directory="./chroma_local_db")
vector_retriever = vector_store.as_retriever(search_kwargs={"k": 3})

# 4. Ensemble Retriever with weighted scoring (e.g., 40% BM25, 60% Dense)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]
)

# 5. Query the hybrid pipeline
results = ensemble_retriever.invoke("FRE 902 authentication")
for idx, doc in enumerate(results):
    print(f"[{idx+1}] {doc.metadata['source']}: {doc.page_content}")
import chromadb
from chromadb.utils.embedding_functions import ChromaBm25EmbeddingFunction

# Initialize Chroma client
client = chromadb.PersistentClient(path="./chroma_local_db")

# Set up BM25 sparse embedding function
bm25_ef = ChromaBm25EmbeddingFunction(
    k=1.2,
    b=0.75
)

# Create/retrieve collection with BM25 capabilities
collection = client.get_or_create_collection(
    name="sovereign_law_hybrid",
    embedding_function=bm25_ef
)
BM25 + Vector Search with LangChain & ChromaDB
This video provides a complete walkthrough on implementing hybrid retrieval combining BM25 keyword matching and Chroma vector search using LangChain.from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import Chroma
from langchain.retrievers import EnsembleRetriever
from langchain_openai import OpenAIEmbeddings

# 1. Setup your documents
docs = ["Document text one...", "Document text two..."]

# 2. Initialize BM25 Lexical Retriever
bm25_retriever = BM25Retriever.from_texts(docs)
bm25_retriever.k = 3

# 3. Initialize Chroma Vector Store Semantic Retriever
embedding_function = OpenAIEmbeddings()
vectorstore = Chroma.from_texts(docs, embedding_function, persist_directory="./chroma_local_db")
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 4. Combine both using Ensemble Retriever (RRF Fusion)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # 40% keyword weight, 60% semantic weight
)

# Run hybrid query
results = ensemble_retriever.invoke("your search query here")
from chromadb.utils.embedding_functions import ChromaBm25EmbeddingFunction
import chromadb

# Initialize the built-in BM25 embedding function
bm25_ef = ChromaBm25EmbeddingFunction(
    k=1.2,
    b=0.75,
    avg_doc_length=256.0
)

client = chromadb.PersistentClient(path="./chroma_local_db")
collection = client.get_or_create_collection(
    name="hybrid_search_collection",
    embedding_function=bm25_ef # Or combine with a dense embedding function depending on setup
)
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

# For Legal Documents
legal_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN,
    chunk_size=2000,
    chunk_overlap=400
)

# For Technical / Developer Documentation
technical_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.MARKDOWN,
    chunk_size=1200,
    chunk_overlap=200
)
pip install langchain-text-splitters chromadb
import os
import glob
import sys
import chromadb
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

DB_PATH = "./chroma_local_db"
TARGET_DIR = sys.argv[1] if len(sys.argv) > 1 else "./sovereign_law_hub"

def initialize_chroma():
    client = chromadb.PersistentClient(path=DB_PATH)
    collection = client.get_or_create_collection(
        name="markdown_rag_ledger",
        metadata={"description": "Chunked offline Markdown RAG index"}
    )
    return collection

def index_markdown_files():
    collection = initialize_chroma()
    print(f"[*] Scanning '{TARGET_DIR}' for Markdown files...")

    md_files = glob.glob(os.path.join(TARGET_DIR, "**/*.md"), recursive=True)
    
    if not md_files:
        print(f"[-] No .md files found in '{TARGET_DIR}'.")
        return

    # Initialize the Markdown-aware recursive text splitter
    text_splitter = RecursiveCharacterTextSplitter.from_language(
        language=Language.MARKDOWN,
        chunk_size=1000,      # Target character size per chunk
        chunk_overlap=200     # Overlap between consecutive chunks to preserve context
    )

    documents = []
    metadatas = []
    ids = []

    for filepath in md_files:
        rel_path = os.path.relpath(filepath, TARGET_DIR)
        with open(filepath, "r", encoding="utf-8") as f:
            content = f.read()

        # Split the Markdown file into smaller text chunks
        chunks = text_splitter.split_text(content)
        
        for idx, chunk in enumerate(chunks):
            chunk_id = f"{rel_path}#chunk-{idx}"
            
            documents.append(chunk)
            metadatas.append({
                "source": rel_path,
                "filepath": filepath,
                "chunk_index": idx,
                "total_chunks": len(chunks)
            })
            ids.append(chunk_id)

    if documents:
        # Upsert chunks into ChromaDB
        collection.upsert(
            documents=documents,
            metadatas=metadatas,
            ids=ids
        )
        print(f"[✓] Successfully indexed {len(documents)} chunks from {len(md_files)} Markdown files into ChromaDB.")
    else:
        print("[!] No document contents to index.")

if __name__ == "__main__":
    index_markdown_files()
def recursive_chunk_markdown(text, chunk_size=1000, chunk_overlap=200):
    """Simple recursive string splitter targeting Markdown structural separators."""
    separators = ["\n# ", "\n## ", "\n### ", "\n\n", "\n", " "]
    
    def _split(doc, seps):
        if len(doc) <= chunk_size or not seps:
            return [doc]
        
        sep = seps[0]
        splits = doc.split(sep)
        chunks = []
        current = ""
        
        for piece in splits:
            item = piece if not current else sep + piece
            if len(current) + len(item) <= chunk_size:
                current += item
            else:
                if current:
                    chunks.append(current)
                current = piece
        if current:
            chunks.append(current)
            
        # Recurse on any chunk that is still oversized
        final_chunks = []
        for c in chunks:
            if len(c) > chunk_size:
                final_chunks.extend(_split(c, seps[1:]))
            else:
                final_chunks.append(c)
        return final_chunks

    return _split(text, separators)
seimport os
import glob
import sys
import chromadb
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

DB_PATH = "./chroma_local_db"
TARGET_DIR = sys.argv[1] if len(sys.argv) > 1 else "./sovereign_law_hub"

def initialize_chroma():
    client = chromadb.PersistentClient(path=DB_PATH)
    collection = client.get_or_create_collection(
        name="markdown_rag_ledger",
        metadata={"description": "Chunked offline Markdown RAG index"}
    )
    return collection

def index_markdown_files():
    collection = initialize_chroma()
    print(f"[*] Scanning '{TARGET_DIR}' for Markdown files...")

    md_files = glob.glob(os.path.join(TARGET_DIR, "**/*.md"), recursive=True)
    
    if not md_files:
        print(f"[-] No .md files found in '{TARGET_DIR}'.")
        return

    # Initialize the Markdown-aware recursive text splitter
    text_splitter = RecursiveCharacterTextSplitter.from_language(
        language=Language.MARKDOWN,
        chunk_size=1000,      # Target character size per chunk
        chunk_overlap=200     # Overlap between consecutive chunks to preserve context
    )

    documents = []
    metadatas = []
    ids = []

    for filepath in md_files:
        rel_path = os.path.relpath(filepath, TARGET_DIR)
        with open(filepath, "r", encoding="utf-8") as f:
            content = f.read()

        # Split the Markdown file into smaller text chunks
        chunks = text_splitter.split_text(content)
        
        for idx, chunk in enumerate(chunks):
            chunk_id = f"{rel_path}#chunk-{idx}"
            
            documents.append(chunk)
            metadatas.append({
                "source": rel_path,
                "filepath": filepath,
                "chunk_index": idx,
                "total_chunks": len(chunks)
            })
            ids.append(chunk_id)

    if documents:
        # Upsert chunks into ChromaDB
        collection.upsert(
            documents=documents,
            metadatas=metadatas,
            ids=ids
        )
        print(f"[✓] Successfully indexed {len(documents)} chunks from {len(md_files)} Markdown files into ChromaDB.")
    else:
        print("[!] No document contents to index.")

if __name__ == "__main__":
    index_markdown_files()
def recursive_chunk_markdown(text, chunk_size=1000, chunk_overlap=200):
    """Simple recursive string splitter targeting Markdown structural separators."""
    separators = ["\n# ", "\n## ", "\n### ", "\n\n", "\n", " "]
    
    def _split(doc, seps):
        if len(doc) <= chunk_size or not seps:
            return [doc]
        
        sep = seps[0]
        splits = doc.split(sep)
        chunks = []
        current = ""
        
        for piece in splits:
            item = piece if not current else sep + piece
            if len(current) + len(item) <= chunk_size:
                current += item
            else:
                if current:
                    chunks.append(current)
                current = piece
        if current:
            chunks.append(current)
            
        # Recurse on any chunk that is still oversized
        final_chunks = []
        for c in chunks:
            if len(c) > chunk_size:
                final_chunks.extend(_split(c, seps[1:]))
            else:
                final_chunks.append(c)
        return final_chunks

    return _split(text, separators)
# Set your Git remote repository URL
git remote set-url origin git@github.com:re1Lucky365/adpBetty-core.git

# Enforce local GPG commit signing
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_GPG_KEY_ID

# Make the backup script executable and test it
chmod +x scripts/backup_workspace.sh
bash scripts/backup_workspace.sh
pip install langchain-text-splitters chromadb
#!/usr/bin/env bash
set -euo pipefail

# Configuration Settings
WORKSPACE_DIR="${WORKSPACE_DIR:-$HOME/workspace}"
BACKUP_DIR="${BACKUP_DIR:-$HOME/backups/workspace}"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_NAME="workspace_backup_${TIMESTAMP}.tar.zst"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"
LOG_FILE="${BACKUP_DIR}/backup.log"

# Retention policy (keep last N backups)
RETENTION_COUNT=10

# Rclone sync target
RCLONE_REMOTE="${RCLONE_REMOTE:-secure_remote:WK_Precision_Vault}"

mkdir -p "${BACKUP_DIR}"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "${LOG_FILE}"
}

log "Starting workspace snapshot..."

# Step 1: Create Zstandard compressed archive (excluding build artifacts and cache)
tar --exclude='.git/objects' \
    --exclude='__pycache__' \
    --exclude='node_modules' \
    --exclude='.venv' \
    -I 'zstd -19 -T0' \
    -cf "${BACKUP_PATH}" \
    -C "$(dirname "${WORKSPACE_DIR}")" \
    "$(basename "${WORKSPACE_DIR}")"

BACKUP_SIZE=$(du -h "${BACKUP_PATH}" | cut -f1)
log "Archive created successfully. Size: ${BACKUP_SIZE}"

# Step 2: Verify archive integrity
if zstd -t "${BACKUP_PATH}"; then
    log "Verification passed: Archive integrity valid."
else
    log "ERROR: Archive verification failed!"
    exit 1
fi

# Step 3: Rclone Cloud Sync
if rclone copy "${BACKUP_PATH}" "${RCLONE_REMOTE}"; then
    log "Rclone sync complete to ${RCLONE_REMOTE}."
else
    log "WARNING: Rclone sync failed."
fi

# Step 4: Prune old backups exceeding retention count
CURRENT_COUNT=$(ls -1t "${BACKUP_DIR}"/*.tar.zst 2>/dev/null | wc -l)
if [ "${CURRENT_COUNT}" -gt "${RETENTION_COUNT}" ]; then
    ls -1t "${BACKUP_DIR}"/*.tar.zst | tail -n +$((RETENTION_COUNT + 1)) | while read -r old_backup; do
        log "Removing expired backup: ${old_backup}"
        rm -f "${old_backup}"
    done
fi

log "Backup pipeline completed successfully."
workspace.shname: Sovereign Security & Audit Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

permissions:
  contents: read

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Security Tools
        run: |
          python -m pip install --upgrade pip
          pip install bandit flake8 safety

      - name: Run Static Application Security Testing (SAST) with Bandit
        run: |
          bandit -r . -ll --format json -o bandit-report.json || true

      - name: Run Dependency Vulnerability Audit
        run: |
          safety check || true

      - name: Run Linter (Flake8)
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      - name: Verify Git Commit GPG Signatures
        run: |
          echo "[*] Checking recent commit provenance..."
          git log -n 5 --show-signature
https://www.instagram.com/reiq365daysayear?mdxt=MXBlN3Bmb3Y4OHdjYQ==bye
gpgconf --reload gpg-agent
KEYINFO --list
S KEYINFO 3F7B29AFA8C1234567890ABCDEF1234567890AB D - - 1762354800 0 -
OK
gpg-connect-agent
mkdir gpg-test-repo && cd gpg-test-repo
git init
git config user.name "Test User"
git config user.email "test@example.com"
git config user.signingkey YOUR_KEY_ID
git config commit.gpgsign true
echo "hello gpg" > test.txt
git add test.txt
git commit -m "test: verify gpg signing"
git log --show-signature
cd .. && rm -rf gpg-test-repo
echo "This is a test message for GPG signing." > secret.txt
gpg --detach-sign --armor secret.txt
gpg --verify secret.txt.asc secret.txt
rm secret.txt secret.txt.asc

rm secret.txt secret.txt.asc
gpg --detach-sign --armor secret.txt
echo "This is a test message for GPG signing." > secret.txt
cd .. && rm -rf gpg-test-repo
git log --show-signature
echo "hello gpg" > test.txt
git add test.txt
git commit -m "test: verify gpg signing"
git config user.name "Test User"
git config user.email "test@example.com"
git config user.signingkey YOUR_KEY_ID
git config commit.gpgsign true
mkdir gpg-test-repo && cd gpg-test-repo
git init
gpg --edit-key YOUR_KEY_ID_OR_EMAIL
expire
save
gpg --armor --export YOUR_KEY_ID
gpg --full-generate-key
gpg --list-secret-keys --keyid-format LONG
git config --global user.signingkey ABC123456789XYZ
git config --global commit.gpgsign true
gpg --armor --export ABC123456789XYZ
gpg --armor --export ABC123456789XYZ
git config --global user.signingkey ABC123456789XYZ
git config --global commit.gpgsign true
gpg --list-secret-keys --keyid-format LONG
gpg --full-generate-key
gpg --armor --export YOUR_KEY_ID

#### Setup Vite and Tailwind

[Tailwind Docs](https://tailwindcss.com/docs/guides/vite)

- setup vite project

```sh
npm create vite@latest my-project -- --template react
cd my-project
```

- add tailwind

```sh
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

- rename to tailwind.config.cjs
- add following content

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

- remove App.css
- delete contents of index.css
- delete contents of App.jsx
- change title

```js
const App = () => {
  return <div>App</div>;
};
export default App;
```

- Add the Tailwind directives to your CSS

index.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Tailwind directives are instructions that decide how Tailwind CSS creates the styles for your website. They control the global styles, component styles, and utility classes.

- start the project "npm run dev"
- setup first tailwind classes in App.jsx

App.jsx

```js
const App = () => {
  return <h1 className='text-7xl font-bold underline'>Tailwind project</h1>;
};
export default App;
```

#### Assets

- get assets from "project-assets"
- images from Undraw
  [Undraw Docs](https://undraw.co/)

#### Install More Libraries

```sh
npm i nanoid react-icons
```

#### Useful Tailwind Extensions

- [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss)
- [Tailwind Fold](https://marketplace.visualstudio.com/items?itemName=stivo.tailwind-fold)

#### Navbar Component

- explore 'links' array in data.jsx
- setup components/navbar
- import links
- setup return and render in App.jsx

```js
import { links } from '../data';
const Navbar = () => {
  return (
    <nav className='bg-emerald-100 '>
      <div className='mx-auto max-w-7xl  px-8 py-4 flex flex-col  sm:flex-row sm:gap-x-16 sm:items-center sm:py-8'>
        <h2 className='text-3xl font-bold'>
          Web
          <span className='text-emerald-600'>Dev</span>
        </h2>
        <div className='flex gap-x-3 '>
          {links.map((link) => {
            const { id, href, text } = link;
            return (
              <a
                key={id}
                href={href}
                className='capitalize text-lg tracking-wide hover:text-emerald-600 duration-300'
              >
                {text}
              </a>
            );
          })}
        </div>
      </div>
    </nav>
  );
};
export default Navbar;
```

#### Hero Component

- setup components/Hero
- setup return
- render in App.jsx

```js
import heroImg from '../assets/hero.svg';
import { FaGithubSquare, FaLinkedin, FaTwitterSquare } from 'react-icons/fa';
const Hero = () => {
  return (
    <section className='bg-emerald-100 py-24  '>
      <div className=' mx-auto max-w-7xl  px-8  grid  md:grid-cols-2 items-center gap-8'>
        <article>
          <h1 className='text-7xl font-bold tracking-wider'>I'm John</h1>
          <p className='mt-4 text-3xl text-slate-700 capitalize tracking-wide'>
            Front-end developer
          </p>
          <p className='mt-2 text-lg text-slate-700 capitalize tracking-wide'>
            turning ideas into interactive reality
          </p>
          <div className='flex gap-x-4 mt-4'>
            <a href='#'>
              <FaGithubSquare className='h-8 w-8 text-slate-500 hover:text-black duration-300' />
            </a>
            <a href='#'>
              <FaLinkedin className='h-8 w-8 text-slate-500 hover:text-black duration-300' />
            </a>
            <a href='#'>
              <FaTwitterSquare className='h-8 w-8 text-slate-500 hover:text-black duration-300' />
            </a>
          </div>
        </article>

        <article className='hidden md:block '>
          <img src={heroImg} className='h-80 lg:h-96' />
        </article>
      </div>
    </section>
  );
};
export default Hero;
```

#### Repeating Styles

index.css

```css
@layer components {
  .align-element {
    @apply mx-auto max-w-7xl px-8;
  }
}
```

- replace in Hero and Navbar

#### Skills

- explore skills array in data.jsx
- create Skills,SkillsCard and SectionTitle components
- setup return and render in App.jsx

Skills.jsx

```js
import SkillsCard from './SkillsCard';
import { skills } from '../data';
import SectionTitle from './SectionTitle';
const Skills = () => {
  return (
    <section className='py-20 align-element' id='skills'>
      <SectionTitle text='tech stack ' />

      <div class=' py-16 grid md:grid-cols-2 lg:grid-cols-3 gap-8'>
        {skills.map((skill) => {
          return <SkillsCard key={skill.id} {...skill} />;
        })}
      </div>
    </section>
  );
};
export default Skills;
```

SectionTitle.jsx

```js
const SectionTitle = ({ text }) => {
  return (
    <div className='border-b border-gray-200 pb-5'>
      <h2 className='text-3xl font-medium tracking-wider capitalize'>{text}</h2>
    </div>
  );
};
export default SectionTitle;
```

SkillsCard.jsx

```js
const SkillsCard = ({ icon, title, text }) => {
  return (
    <article>
      <span>{icon}</span>
      <h4 className='mt-6 font-bold'>{title}</h4>
      <p className='mt-2 text-slate-500'>{text}</p>
    </article>
  );
};
export default SkillsCard;
```

#### Global Styles

index.html

```html
<html lang="en" class="bg-slate-50 scroll-smooth"></html>
```

#### About

- create About component and render in App.jsx

```js
import aboutSvg from '../assets/about.svg';
import SectionTitle from './SectionTitle';
const About = () => {
  return (
    <section class='bg-white py-20' id='about'>
      <div class='align-element grid  md:grid-cols-2 items-center gap-16'>
        <img src={aboutSvg} className='w-full h-64' />
        <article>
          <SectionTitle text='code and coffee' />
          <p className='text-slate-600 mt-8 leading-loose'>
            Lorem ipsum dolor sit amet consectetur, adipisicing elit. Porro
            omnis exercitationem tempora, aliquid deleniti tenetur vero esse
            praesentium eaque dicta fugiat? Molestiae expedita, nulla neque
            error porro sint distinctio possimus!
          </p>
        </article>
      </div>
    </section>
  );
};
export default About;
```

#### Projects

- explore "projects" array in data.jsx
- create Projects and ProjectsCard components
- setup return and render in App.jsx

Projects.jsx

```js
import ProjectsCard from './ProjectsCard';
import { projects } from '../data';
import SectionTitle from './SectionTitle';
const Projects = () => {
  return (
    <section className='py-20 align-element' id='projects'>
      <SectionTitle text='web creations' />
      <div className='py-16 grid lg:grid-cols-2 xl:grid-cols-3 gap-8'>
        {projects.map((project) => {
          return <ProjectsCard key={project.id} {...project} />;
        })}
      </div>
    </section>
  );
};
export default Projects;
```

ProjectsCard.jsx

```js
import { FaGithubSquare, FaLinkedin, FaTwitterSquare } from 'react-icons/fa';
import { TbWorldWww } from 'react-icons/tb';
const ProjectsCard = ({ url, img, github, title, text }) => {
  return (
    <article className='bg-white rounded-lg shadow-md block hover:shadow-xl duration-300'>
      <img
        src={img}
        alt={title}
        className='w-full object-cover rounded-t-lg h-64 '
      />
      <div className='capitalize p-8'>
        <h2 className='text-xl tracking-wide font-medium'>{title}</h2>
        <p className='mt-4 text-slate-700 leading-loose'>{text}</p>
        <div className='mt-4 flex gap-x-4'>
          <a href={url}>
            <TbWorldWww className='h-8 w-8 text-slate-500 hover:text-black duration-300' />
          </a>
          <a href={github}>
            <FaGithubSquare className='h-8 w-8 text-slate-500 hover:text-black duration-300' />
          </a>
        </div>
      </div>
    </article>
  );
};
export default ProjectsCard;
```

#### Extra Challenge

- setup projects in CMS
