# Setting up ArchiveBox locally and on VPS with Sync

This is a quick note about how to install [ArchiveBox](https://github.com/ArchiveBox/) on a cloud VPS (like Hetzner vCloud), and synchronize it with one or more local machines using a robust `rsync` and `archivebox init` workflow. This setup provides offline capability and avoids database conflicts.

This tutorial documents a real-world troubleshooting process, solving common issues like permission errors, missing dependencies, and IP blocking, culminating in a flexible multi-master sync solution.

## Architecture Overview

This sets up the following system:

```
┌──────────────────┐      ┌─────────────────────────┐      ┌──────────────────┐
│  Local Machine   │      │      Cloud VPS Server   │      │  Other Devices   │
│ (Mac/Win/Linux)  │      │       (Hetzner etc.)    │      │   (iOS/Android)  │
│                  │◄─────┼────────► rsync ◄────────┼─────►│                  │
│  • Writable DB   │      │      • Writable DB      │      │  • Web Access    │
│  • Add any URL   │      │      • Add simple URLs  │      │                  │
│  • Browser Ext.  │      │      • 24/7 Web UI      │      │                  │
│  • Offline Work  │      │      • Central Backup   │      │                  │
└──────────────────┘      └─────────────────────────┘      └──────────────────┘
```

### Key Benefits of this Setup
* **Offline Capability:** Add and manage links on your local machine without an internet connection.
* **No Database Conflicts:** The `archivebox init` method safely merges archives without corrupting the database, which is a major risk with other sync methods like `git`.
* **Efficient Transfer:** `rsync` is used to only copy changed files, saving time and bandwidth.
* **Handles All Websites:** Archive difficult sites (like `archive.ph` or those with bot detection) on your local machine using its trusted residential IP, then sync the successful result to the server.
* **Flexible Workflow:** Add links on any machine and use the sync script to reconcile the collections.

---

## Step 1: Create a Dedicated User on the VPS

First, create a non-root user specifically for ArchiveBox to enhance security.

```bash
# As root on your VPS
sudo useradd -m -s /bin/bash archivebox

# Set a strong password for this user
sudo passwd archivebox
```

## Step 2: Install System Dependencies on VPS

Install Python, `pip`, `git`, `rsync`, and the browser dependencies needed for headless Chrome/Chromium to run.

```bash
# As root on your VPS
sudo apt update
sudo apt install -y python3-pip python3-venv git rsync nodejs npm

# Install browser libraries needed by Playwright/Chromium
sudo apt install -y libatk1.0-0 libatk-bridge2.0-0 libcups2 libxkbcommon0 \
    libatspi2.0-0 libxcomposite1 libxdamage1 libxext6 libxfixes3 libxrandr2 \
    libgbm1 libpango-1.0-0 libcairo2 libasound2
```

## Step 3: Install and Initialize ArchiveBox on VPS

We will perform all subsequent actions as the new `archivebox` user.

```bash
# Switch to the new user
su - archivebox

# Install ArchiveBox for this user only
pip3 install --user 'archivebox[sonic,ripgrep]'

# Add the user's local bin directory to the PATH for the current session and for future logins
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Create the data directory and initialize ArchiveBox
mkdir -p ~/.local/share/archivebox
cd ~/.local/share/archivebox
archivebox init --setup

# Create an admin user for the web interface
archivebox manage createsuperuser
# Follow the prompts to set a username, email, and password
```

## Step 4: Configure ArchiveBox on VPS

Let's configure the server instance for security and performance.

```bash
# Make sure you are the archivebox user in the data directory
# cd ~/.local/share/archivebox

# Disable all public access
archivebox config --set PUBLIC_INDEX=False
archivebox config --set PUBLIC_SNAPSHOTS=False
archivebox config --set PUBLIC_ADD_VIEW=False

# Recommended settings for a server
archivebox config --set CHROME_HEADLESS=True
archivebox config --set CHROME_SANDBOX=False
archivebox config --set TIMEOUT=120
archivebox config --set SNAPSHOTS_PER_PAGE=40
archivebox config --set FOOTER_INFO="Private Family Archive"
```

## Step 5: Create a Systemd Service on VPS

A systemd service will ensure ArchiveBox starts automatically on boot and runs continuously in the background.

```bash
# Exit from the archivebox user back to root
exit

# Create the systemd service file
sudo tee /etc/systemd/system/archivebox.service << 'EOF'
[Unit]
Description=ArchiveBox Web Server
After=network.target

[Service]
Type=simple
User=archivebox
Group=archivebox
WorkingDirectory=/home/archivebox/.local/share/archivebox
# The PATH must include the user's local bin directory
Environment="PATH=/home/archivebox/.local/bin:/usr/bin:/bin"
ExecStart=/home/archivebox/.local/bin/archivebox server 0.0.0.0:8080
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable, start, and check the status of the service
sudo systemctl enable archivebox
sudo systemctl start archivebox
sudo systemctl status archivebox
```

## Step 6: Configure Firewall on VPS

If you use a firewall like `ufw`, you must allow traffic to the port ArchiveBox is using.

```bash
# As root on your VPS
sudo ufw allow 8080/tcp
```

You should now be able to access your ArchiveBox instance at `http://YOUR_VPS_IP:8080`.

## Step 7: Set Up Your Local Machine

Repeat a similar installation process on your local machine (e.g., your MacBook).

```bash
# Example for macOS with Homebrew
brew install archivebox
brew install rsync # macOS's default rsync can be outdated

# Or for any OS with pip
pip3 install 'archivebox[sonic,ripgrep]'

# Create and initialize your local data directory
mkdir -p ~/archivebox/data
cd ~/archivebox/data
archivebox init --setup
```
Configure your local instance as you see fit. You can use the same `archivebox config` commands as in Step 4.

## Step 8: Set Up SSH Key Authentication

Passwordless SSH access is essential for the sync script to work smoothly.

1.  **On your local machine**, generate an SSH key if you don't have one:
    ```bash
    ssh-keygen -t rsa -b 4096
    ```
2.  **Copy the key to your VPS** for the `archivebox` user:
    ```bash
    ssh-copy-id archivebox@YOUR_VPS_IP
    ```
3.  **Test the connection:**
    ```bash
    ssh archivebox@YOUR_VPS_IP "echo 'Connection successful!'"
    ```

## Step 9: Create the Master Sync Script

This script is the heart of the workflow. Save it on your local machine.

1.  **Create the script file** on your local machine:
    ```bash
    # This command creates the script in your home directory
    cat > ~/sync_archivebox.sh << 'EOF'
    #!/bin/bash

    # ==> Generic ArchiveBox Two-Way Sync Script <==
    #
    # This script synchronizes an ArchiveBox collection between a local machine
    # and a remote cloud server, allowing for a multi-master workflow.
    # It uses rsync for efficient file transfer and `archivebox init` to
    # safely merge the database indexes without conflicts.
    #
    
    # --- Configuration ---
    # ==> EDIT THE VALUES IN THIS SECTION <==
    
    # The path to your local ArchiveBox data directory.
    # Using $HOME makes it portable for any user.
    LOCAL_DIR="$HOME/archivebox/data"
    
    # Details for your Cloud server
    REMOTE_USER="archivebox"
    REMOTE_HOST="YOUR_VPS_IP_OR_HOSTNAME"
    REMOTE_DIR="/home/archivebox/.local/share/archivebox"
    REMOTE_ARCHIVEBOX_BINARY="/home/archivebox/.local/bin/archivebox"
    
    # --- End Configuration ---
    
    
    # This function pushes your local changes up to the server
    push_to_server() {
        echo "▶️  Pushing local changes to Cloud server..."
    
        # Step 1: Sync the archive files and URL sources UP to the server.
        echo "    -> Step 1/2: Syncing archive files..."
        rsync -avz --info=progress2 "$LOCAL_DIR/archive/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/archive/"
        rsync -avz --info=progress2 "$LOCAL_DIR/sources/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/sources/"
    
        # Step 2: Trigger 'archivebox init' on the server to rebuild its index.
        # The "ssh -t" command forces a terminal to be allocated, showing full, colorful output.
        echo "    -> Step 2/2: Rebuilding server index..."
        ssh -t "$REMOTE_USER@$REMOTE_HOST" "cd $REMOTE_DIR && $REMOTE_ARCHIVEBOX_BINARY init"
    
        echo "✅ Push complete. Server is now synced."
    }
    
    # This function pulls the server's changes down to your local machine
    pull_from_server() {
        echo "🔽 Pulling remote changes from Cloud server..."
    
        # Step 1: Sync the archive files and URL sources DOWN from the server.
        echo "    -> Step 1/2: Syncing archive files..."
        rsync -avz --info=progress2 "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/archive/" "$LOCAL_DIR/archive/"
        rsync -avz --info=progress2 "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/sources/" "$LOCAL_DIR/sources/"
    
        # Step 2: Trigger 'archivebox init' on your local device to rebuild the local index.
        echo "    -> Step 2/2: Rebuilding local index..."
        cd "$LOCAL_DIR" && archivebox init
    
        echo "✅ Pull complete. Local machine is now synced."
    }
    
    # --- Main Script Logic ---
    if [[ "$1" == "push" ]]; then
        push_to_server
    elif [[ "$1" == "pull" ]]; then
        pull_from_server
    else
        echo "Usage: $0 [push|pull]"
        echo "  push: Sync your local changes UP to the Cloud server."
        echo "  pull: Sync the Cloud server's changes DOWN to your local machine."
        exit 1
    fi
    EOF
    ```

2.  **IMPORTANT: Edit the script** with your server's IP address.
    ```bash
    # Replace the placeholder with your actual IP. Example:
    sed -i.bak 's/YOUR_VPS_IP/123.231.23.31/' ~/sync_archivebox.sh
    ```

3.  **Make the script executable:**
    ```bash
    chmod +x ~/sync_archivebox.sh
    ```

## Step 10: Daily Usage Workflow

Your setup is complete. Here is how you use it.

* **To sync changes from your local device UP to the server:**
    ```bash
    ~/sync_archivebox.sh push
    ```

* **To sync changes from the server DOWN to your local device:**
    ```bash
    ~/sync_archivebox.sh pull
    ```

Always `pull` before you start adding new links locally if you suspect changes have been made on the server. If you only ever add links on your Mac, you will only ever need to `push`.

Of course. Here are the final, fully-debugged scripts, written in a generic form with clear placeholders, ready to be added to your markdown tutorial.

This new section can be added after "Step 10: Daily Usage Workflow" in your guide.

---

## Bonus: Creating a Daily EPUB Digest for Reading

To enhance the reading experience, you can automatically generate a daily "newspaper" of your recently saved articles in EPUB format. This single file will contain a linked table of contents and all your articles, perfect for e-readers or tablets.

This process requires **Pandoc**, a universal document converter.

### Step 1: Install Dependencies on the VPS

First, log in to your VPS as `root` and install Pandoc.

```bash
# As root on your VPS
sudo apt update
sudo apt install -y pandoc
```

### Step 2: Create the Digest Script on the VPS

This script runs on your server to find recent articles and use Pandoc to package them into a single EPUB file.

1.  **Log in as the `archivebox` user:**
    ```bash
    su - archivebox
    ```
2.  **Create the script file:**
    ```bash
    # This command creates the script in the archivebox user's home directory
    cat > ~/create_digest_epub.sh << 'EOF'
    #!/bin/bash

    # --- Configuration ---
    ARCHIVEBOX_DIR="/home/archivebox/.local/share/archivebox"
    EXPORT_DIR="/home/archivebox/epub_exports"
    
    # Default to creating a digest of the last 1 day.
    # This can be overridden by passing a number as an argument, e.g., ./create_digest_epub.sh 7
    DAYS_AGO=${1:-1}

    # --- Script ---
    echo "▶️  Starting digest creation process for the last $DAYS_AGO day(s)."
    mkdir -p "$EXPORT_DIR"

    # Create a temporary directory that will be automatically cleaned up on exit
    WORK_DIR=$(mktemp -d)
    trap 'rm -rf "$WORK_DIR"' EXIT

    COVER_FILE="$WORK_DIR/00_cover.html"
    ARTICLE_COUNTER=0
    ARTICLE_LINKS=""
    declare -a ARTICLE_FILES

    echo "▶️  Finding and collecting recent articles..."

    CUTOFF_TIMESTAMP=$(date +%s -d "$DAYS_AGO days ago")
    echo "DEBUG: Looking for snapshots with a timestamp newer than $CUTOFF_TIMESTAMP"

    # Loop through snapshot directories using process substitution to preserve variable scope
    while IFS= read -r -d $'\0' SNAPSHOT_DIR; do
        
        ((TOTAL_CHECKED++))
        SNAPSHOT_TIMESTAMP=$(basename "$SNAPSHOT_DIR")
        # Truncate timestamp to an integer to handle floating-point folder names
        SNAPSHOT_INT=${SNAPSHOT_TIMESTAMP%.*}
        
        # The correct path to the clean article content
        SOURCE_HTML="$SNAPSHOT_DIR/readability/content.html"

        echo -n "DEBUG: Checking Snapshot $SNAPSHOT_TIMESTAMP... "

        if (( SNAPSHOT_INT > CUTOFF_TIMESTAMP )); then
            if [ -f "$SOURCE_HTML" ]; then
                echo "[INCLUDED]"
                ((ARTICLE_COUNTER++))
                
                FILENAME=$(printf "%02d" $ARTICLE_COUNTER).html
                FILEPATH="$WORK_DIR/$FILENAME"
                cp "$SOURCE_HTML" "$FILEPATH"
                
                TITLE=$(python3 -c "import sys, json; print(json.load(open(sys.argv[1]))['title'])" "$SNAPSHOT_DIR/index.json" 2>/dev/null || echo "Untitled Article")
                
                # Prepend the correct H1 title to the file for a clean Table of Contents
                TEMP_FILE="$WORK_DIR/temp.html"
                echo "<h1>$TITLE</h1>" > "$TEMP_FILE"
                cat "$FILEPATH" >> "$TEMP_FILE"
                mv "$TEMP_FILE" "$FILEPATH"
                
                ARTICLE_LINKS+="<li><a href=\"$FILENAME\">$TITLE</a></li>\n"
                ARTICLE_FILES+=("$FILEPATH")
            else
                echo "[SKIPPED - No readability/content.html]"
            fi
        else
            echo "[SKIPPED - Too old]"
        fi
    done < <(find "$ARCHIVEBOX_DIR/archive/" -maxdepth 1 -mindepth 1 -type d -print0)

    echo "▶️  Finished search. Checked $TOTAL_CHECKED total snapshots."

    if [ "$ARTICLE_COUNTER" -eq 0 ]; then
        echo "✅ No new articles found for the digest. Exiting."
        exit 0
    fi

    echo "✅ Found $ARTICLE_COUNTER articles to include in the digest."

    DIGEST_DATE=$(date +"%A, %B %d, %Y")
    
    # Create a clean cover page with only one H1 heading
    {
        echo "<!DOCTYPE html><html><head><title>Digest</title></head><body>"
        echo "<h1>Daily Digest</h1>"
        echo "<p><b>$DIGEST_DATE</b></p>"
        echo "<hr>"
        echo "<p><b>Articles:</b></p>"
        echo "<ul>"
        echo -e "$ARTICLE_LINKS"
        echo "</ul>"
        echo "</body></html>"
    } > "$COVER_FILE"

    FINAL_EPUB_FILENAME="Digest_$(date +%Y-%m-%d)_${DAYS_AGO}days.epub"
    FINAL_EPUB_PATH="$EXPORT_DIR/$FINAL_EPUB_FILENAME"

    echo "▶️  Assembling the final EPUB with Pandoc..."

    # Use --toc-depth=1 to only include H1 tags in the EPUB's Table of Contents
    pandoc \
        --from=html \
        --to=epub \
        --table-of-contents \
        --toc-depth=1 \
        --metadata title="Digest ($DAYS_AGO days): $DIGEST_DATE" \
        --metadata creator="ArchiveBox" \
        --output="$FINAL_EPUB_PATH" \
        "$COVER_FILE" "${ARTICLE_FILES[@]}"

    if [ -f "$FINAL_EPUB_PATH" ]; then
        echo "✅ Success! Digest created at: $FINAL_EPUB_PATH"
    else
        echo "❌ Error: Pandoc failed to create the EPUB."
        exit 1
    fi

    exit 0
    EOF
    ```
3.  **Make the script executable:**
    ```bash
    chmod +x ~/create_digest_epub.sh
    ```

### Step 3: Automate the Digest Creation on the VPS

Set up a cron job to run this script automatically every day.

1.  **As the `archivebox` user**, open the cron table for editing:
    ```bash
    crontab -e
    ```
2.  **Add the following line** to the file to run the script every morning at 5 AM. Save and exit the editor.
    ```
    0 5 * * * /home/archivebox/create_digest_epub.sh > /tmp/digest_creation.log 2>&1
    ```

### Step 4: Create the Sync Script on Your Local Machine

This script, to be saved on your local device, will download the generated EPUBs from the server. This version includes a fix for the outdated `rsync` on macOS.

1.  **On your local device (e.g. MacBook) terminal**, create the script file:
    ```bash
    cat > ~/get_epubs.sh << 'EOF'
    #!/bin/bash

    # --- Configuration ---
    LOCAL_EPUB_DIR="$HOME/Downloads/EPUBS_from_ArchiveBox"
    REMOTE_USER="archivebox"
    REMOTE_HOST="YOUR_VPS_IP" # <-- IMPORTANT: EDIT THIS LINE
    REMOTE_EPUB_DIR="/home/archivebox/epub_exports"
    # --- End Configuration ---

    # --- Auto-detect the best rsync command on macOS ---
    RSYNC_CMD="rsync"
    PROGRESS_FLAG="-v"

    # Check for modern Homebrew rsync and use it if available
    if [ -f "/opt/homebrew/bin/rsync" ]; then
        RSYNC_CMD="/opt/homebrew/bin/rsync"
        PROGRESS_FLAG="--info=progress2"
    elif [ -f "/usr/local/bin/rsync" ]; then
        RSYNC_CMD="/usr/local/bin/rsync"
        PROGRESS_FLAG="--info=progress2"
    fi

    echo "▶️  Using rsync at: $RSYNC_CMD"
    echo "🔽 Downloading new EPUB digests from the server..."

    mkdir -p "$LOCAL_EPUB_DIR"

    # Use the detected rsync command and progress flag
    "$RSYNC_CMD" -az "$PROGRESS_FLAG" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_EPUB_DIR/" "$LOCAL_EPUB_DIR/"

    echo "✅ Sync complete. New EPUBs are in $LOCAL_EPUB_DIR"
    open "$LOCAL_EPUB_DIR"
    EOF
    ```
2.  **IMPORTANT: Edit the script** with your server's IP address.
    ```bash
    # Replace the placeholder with your actual IP. For example:
    sed -i.bak 's/YOUR_VPS_IP/123.231.23.21/' ~/get_epubs.sh
    ```
3.  **Make the script executable:**
    ```bash
    chmod +x ~/get_epubs.sh
    ```

### Step 5: Daily Usage

Whenever you want to download the latest digest, simply run the script on your local device:

```bash
~/get_epubs.sh
```

## Troubleshooting Appendix

If you encounter issues, here are some hints.

* **Permission Issues on VPS:** If you get a `Permission denied` error, a file was likely edited by `root`. Fix it by resetting ownership.
    ```bash
    # Run as root on the VPS
    chown -R archivebox:archivebox /home/archivebox/.local/share/archivebox
    ```

* **Missing Dependencies on VPS:** If archiving methods fail, run the setup again.
    ```bash
    # Run as the archivebox user on the VPS
    archivebox setup
    ```
* **Command Not Found on VPS:** You are likely logged in as `root`. Switch to the correct user.
    ```bash
    su - archivebox
    ```
* **Service Issues on VPS:** To check the status or restart the background server process:
    ```bash
    # Run as root on the VPS
    sudo systemctl status archivebox
    sudo systemctl restart archivebox
    ```
