# ArchiveBox ↔ Wallabag Bidirectional Sync Tutorial

**A comprehensive guide to setting up a self-hosted, bidirectional article archiving and reading system**

## Architecture Overview

This tutorial creates a powerful three-tier bidirectional sync system that allows you to save articles from mobile devices and have them automatically archived across multiple locations:

```
┌──────────────────┐    ┌─────────────────────────┐    ┌──────────────────┐
│  Local Machine   │    │      Cloud VPS Server   │    │    Home NAS      │
│ (Mac/Win/Linux)  │    │       (Hetzner etc.)    │    │   (Home Network) │
│                  │    │                         │    │                  │
│  • ArchiveBox    │◄──►│  • ArchiveBox (Master)  │◄──►│  • Wallabag      │
│  • Full Archive  │    │  • Full Archive         │    │  • Reading UI    │
│  • Offline Work  │    │  • Wallabag→AB Sync     │    │  • URL Collection│
└──────────────────┘    └─────────────────────────┘    └──────────────────┘
                                                               ▲
                                                               │
                                                        ┌──────────────┐
                                                        │ Mobile Apps  │
                                                        │ (iOS/Android)│
                                                        │ • Wallabag   │
                                                        │ • Save URLs  │
                                                        └──────────────┘
```

## System Benefits

✅ **True bidirectional sync**: URLs flow both ways automatically  
✅ **Mobile integration**: Save from phone → automatically archived  
✅ **Automatic operation**: VPS handles mobile captures every 30 minutes  
✅ **Redundancy**: Content exists in multiple locations  
✅ **Offline capability**: All systems work offline  
✅ **Family friendly**: Beautiful reading interface  
✅ **Self-hosted privacy**: All data stays under your control  
✅ **Efficient transfer**: Only changed files are synced  
✅ **No database conflicts**: Safe merging with ArchiveBox init  

## Prerequisites

- **Local Machine**: macOS/Linux with ArchiveBox installed
- **VPS Server**: Cloud server (Hetzner, DigitalOcean, etc.) with ArchiveBox installed
- **Home NAS**: QNAP, Synology, or any system with Docker support
- **SSH Key Access**: Passwordless SSH between local machine and VPS

## Part 1: NAS Wallabag Setup

### 1.1 Install PostgreSQL Database

```bash
# SSH into your NAS as admin
ssh admin@YOUR_NAS_HOSTNAME

# Create network for wallabag services
sudo docker network create wallabag-network

# Create database directory with proper permissions
sudo mkdir -p /path/to/your/storage/wallabag-db
sudo chown -R 999:999 /path/to/your/storage/wallabag-db

# Run PostgreSQL database
sudo docker run -d \
  --name wallabag-db \
  --restart unless-stopped \
  --network wallabag-network \
  -v /path/to/your/storage/wallabag-db:/var/lib/postgresql/data \
  -e POSTGRES_DB=wallabag \
  -e POSTGRES_USER=wallabag \
  -e POSTGRES_PASSWORD=your_secure_db_password \
  postgres:13

# Wait for database to start
sleep 15
```

### 1.2 Install Wallabag

```bash
# Create wallabag directories
sudo mkdir -p /path/to/your/storage/wallabag/{data,images}
sudo chown -R 1000:1000 /path/to/your/storage/wallabag/

# Run Wallabag with PostgreSQL
sudo docker run -d \
  --name wallabag \
  --restart unless-stopped \
  --network wallabag-network \
  -p 8082:80 \
  -v /path/to/your/storage/wallabag/data:/var/www/wallabag/data \
  -v /path/to/your/storage/wallabag/images:/var/www/wallabag/web/assets/images \
  -e SYMFONY__ENV__DOMAIN_NAME="http://YOUR_NAS_HOSTNAME:8082" \
  -e SYMFONY__ENV__DATABASE_DRIVER=pdo_pgsql \
  -e SYMFONY__ENV__DATABASE_HOST=wallabag-db \
  -e SYMFONY__ENV__DATABASE_PORT=5432 \
  -e SYMFONY__ENV__DATABASE_NAME=wallabag \
  -e SYMFONY__ENV__DATABASE_USER=wallabag \
  -e SYMFONY__ENV__DATABASE_PASSWORD=your_secure_db_password \
  wallabag/wallabag:latest

# Monitor startup - wait for "wallabag is ready!" message
sudo docker logs -f wallabag
```

### 1.3 Initialize Wallabag

```bash
# Enter container to run setup
sudo docker exec -it wallabag sh

# Run the installer
php bin/console wallabag:install --env=prod

# Follow prompts to create admin user:
# Username: your_admin_username
# Password: your_secure_password
# Email: your_email@domain.com

# Clear cache and exit
php bin/console cache:clear --env=prod
exit
```

### 1.4 Set Up API Access

1. **Access Wallabag** at `http://YOUR_NAS_HOSTNAME:8082`
2. **Log in** with your admin credentials
3. **Go to Settings** → **API clients management**
4. **Click "Create a new client"**
5. **Note your credentials**:
   - **Client ID**: `your_client_id_here`
   - **Client Secret**: `your_client_secret_here`

### 1.5 Test API

```bash
# Test authentication (replace with your actual credentials)
TOKEN=$(curl -s -X POST "http://YOUR_NAS_HOSTNAME:8082/oauth/v2/token" \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "client_id": "your_client_id_here",
    "client_secret": "your_client_secret_here",
    "username": "your_admin_username",
    "password": "your_secure_password"
  }' | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# Test adding an article
curl -X POST "http://YOUR_NAS_HOSTNAME:8082/api/entries.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://en.wikipedia.org/wiki/Test", "tags": "api-test"}'
```

## Part 2: VPS ArchiveBox Setup

### 2.1 Create Reverse Sync Script on VPS

```bash
# SSH to your VPS
ssh your_vps_user@YOUR_VPS_IP

# Create the reverse sync script
cat > /home/your_vps_user/wallabag_to_archivebox_sync.sh << 'EOF'
#!/bin/bash

# Wallabag to ArchiveBox Reverse Sync Script
# Runs on VPS to pull new URLs from Wallabag and archive them

# --- Configuration - UPDATE THESE VALUES ---
WALLABAG_URL="http://YOUR_NAS_HOSTNAME:8082"
WALLABAG_CLIENT_ID="your_client_id_here"
WALLABAG_CLIENT_SECRET="your_client_secret_here"
WALLABAG_USERNAME="your_admin_username"
WALLABAG_PASSWORD="your_secure_password"

ARCHIVEBOX_DIR="/path/to/your/archivebox/data"
ARCHIVEBOX_BIN="/path/to/archivebox/binary"
SYNC_STATE_FILE="/home/your_vps_user/wallabag_sync_state.json"
LOG_FILE="/home/your_vps_user/reverse_sync.log"

# --- Functions ---

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

get_wallabag_token() {
    local response=$(curl -s --connect-timeout 10 --max-time 30 \
        -X POST "$WALLABAG_URL/oauth/v2/token" \
        -H "Content-Type: application/json" \
        -d "{
            \"grant_type\": \"password\",
            \"client_id\": \"$WALLABAG_CLIENT_ID\",
            \"client_secret\": \"$WALLABAG_CLIENT_SECRET\",
            \"username\": \"$WALLABAG_USERNAME\",
            \"password\": \"$WALLABAG_PASSWORD\"
        }" 2>/dev/null)
    
    if [ $? -eq 0 ] && [ -n "$response" ]; then
        echo "$response" | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    print(data.get('access_token', ''))
except:
    print('')
" 2>/dev/null
    else
        echo ""
    fi
}

get_wallabag_entries() {
    local token="$1"
    local since_id="$2"
    
    local url="$WALLABAG_URL/api/entries.json?perPage=100&order=desc&sort=created"
    if [ -n "$since_id" ] && [ "$since_id" != "0" ]; then
        url="${url}&since=${since_id}"
    fi
    
    curl -s --connect-timeout 10 --max-time 60 \
        -H "Authorization: Bearer $token" \
        "$url" 2>/dev/null
}

load_sync_state() {
    if [ -f "$SYNC_STATE_FILE" ]; then
        python3 -c "
import json
try:
    with open('$SYNC_STATE_FILE', 'r') as f:
        data = json.load(f)
        print(data.get('last_synced_id', 0))
except:
    print(0)
" 2>/dev/null
    else
        echo "0"
    fi
}

save_sync_state() {
    local last_id="$1"
    python3 -c "
import json
data = {'last_synced_id': $last_id, 'last_sync_time': '$(date -Iseconds)'}
with open('$SYNC_STATE_FILE', 'w') as f:
    json.dump(data, f, indent=2)
" 2>/dev/null
}

process_entries() {
    local entries_json="$1"
    local urls_to_archive=()
    local highest_id=0
    
    # Extract URLs and find highest ID (exclude items already tagged with archivebox-sync)
    while IFS='|' read -r entry_id url title; do
        if [ -n "$url" ] && [ "$url" != "null" ] && [ -n "$entry_id" ]; then
            urls_to_archive+=("$url")
            if [ "$entry_id" -gt "$highest_id" ]; then
                highest_id="$entry_id"
            fi
            log_message "📥 Found: $title ($url)"
        fi
    done < <(echo "$entries_json" | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    entries = data.get('_embedded', {}).get('items', [])
    for entry in entries:
        entry_id = entry.get('id', 0)
        url = entry.get('url', '')
        title = entry.get('title', 'Untitled')
        # Only process entries that don't have archivebox-sync tag to avoid loops
        tags = [tag.get('label', '') for tag in entry.get('tags', [])]
        if 'archivebox-sync' not in tags:
            print(f'{entry_id}|{url}|{title}')
except Exception as e:
    pass
" 2>/dev/null)
    
    # Archive URLs if any found
    if [ ${#urls_to_archive[@]} -gt 0 ]; then
        log_message "🏛️  Archiving ${#urls_to_archive[@]} URLs with ArchiveBox..."
        
        cd "$ARCHIVEBOX_DIR"
        
        # Create temporary file with URLs
        local temp_urls_file=$(mktemp)
        printf '%s\n' "${urls_to_archive[@]}" > "$temp_urls_file"
        
        # Archive the URLs
        "$ARCHIVEBOX_BIN" add --depth=1 < "$temp_urls_file" >> "$LOG_FILE" 2>&1
        local archive_result=$?
        
        rm "$temp_urls_file"
        
        if [ $archive_result -eq 0 ]; then
            log_message "✅ Successfully archived ${#urls_to_archive[@]} URLs"
            echo "$highest_id"
        else
            log_message "❌ ArchiveBox command failed"
            echo "0"
        fi
    else
        log_message "ℹ️  No new URLs to archive"
        echo "$highest_id"
    fi
}

# --- Main Script ---

log_message "🔄 Starting Wallabag → ArchiveBox reverse sync"

# Check if NAS/Wallabag is reachable
if ! curl -s --connect-timeout 5 --max-time 10 "$WALLABAG_URL/api/info" > /dev/null 2>&1; then
    log_message "⚠️  NAS/Wallabag not reachable, skipping sync"
    exit 0
fi

# Get authentication token
log_message "🔑 Authenticating with Wallabag..."
TOKEN=$(get_wallabag_token)

if [ -z "$TOKEN" ]; then
    log_message "❌ Failed to authenticate with Wallabag"
    exit 1
fi

# Load last sync state
LAST_SYNCED_ID=$(load_sync_state)
log_message "📋 Last synced ID: $LAST_SYNCED_ID"

# Get new entries
log_message "📥 Fetching new entries from Wallabag..."
ENTRIES_JSON=$(get_wallabag_entries "$TOKEN" "$LAST_SYNCED_ID")

if [ -z "$ENTRIES_JSON" ]; then
    log_message "❌ Failed to fetch entries from Wallabag"
    exit 1
fi

# Process entries and archive
NEW_HIGHEST_ID=$(process_entries "$ENTRIES_JSON")

# Update sync state if we processed anything
if [ -n "$NEW_HIGHEST_ID" ] && [ "$NEW_HIGHEST_ID" -gt "$LAST_SYNCED_ID" ]; then
    save_sync_state "$NEW_HIGHEST_ID"
    log_message "💾 Updated sync state to ID: $NEW_HIGHEST_ID"
fi

log_message "✅ Reverse sync completed successfully"
EOF

# Make executable and update configuration
chmod +x /home/your_vps_user/wallabag_to_archivebox_sync.sh

# IMPORTANT: Edit the script to update all the configuration variables
# Replace YOUR_NAS_HOSTNAME, credentials, and paths with your actual values
```

### 2.2 Set Up Automated Reverse Sync

```bash
# Add cron job for automatic sync every 30 minutes
(crontab -l 2>/dev/null; echo '*/30 * * * * /home/your_vps_user/wallabag_to_archivebox_sync.sh >> /tmp/reverse_sync_cron.log 2>&1') | crontab -

# Verify cron job
crontab -l

# Test the script manually
/home/your_vps_user/wallabag_to_archivebox_sync.sh
```

## Part 3: Local Machine Setup

### 3.1 Create Enhanced Bidirectional Sync Script

```bash
# Save this script as ~/sync_archivebox_bidirectional.sh
cat > ~/sync_archivebox_bidirectional.sh << 'EOF'
#!/bin/bash

# Enhanced ArchiveBox ↔ Wallabag Bidirectional Sync Script
# Handles both directions: ArchiveBox → Wallabag and Wallabag → ArchiveBox

# --- Configuration - UPDATE THESE VALUES ---
LOCAL_DIR="$HOME/path/to/your/local/archivebox/data"
REMOTE_USER="your_vps_user"
REMOTE_HOST="YOUR_VPS_IP"
REMOTE_DIR="/path/to/your/archivebox/data"
REMOTE_ARCHIVEBOX_BINARY="/path/to/archivebox/binary"
REVERSE_SYNC_SCRIPT="/home/your_vps_user/wallabag_to_archivebox_sync.sh"

WALLABAG_URL="http://YOUR_NAS_HOSTNAME:8082"
WALLABAG_CLIENT_ID="your_client_id_here"
WALLABAG_CLIENT_SECRET="your_client_secret_here"
WALLABAG_USERNAME="your_admin_username"
WALLABAG_PASSWORD="your_secure_password"

DAYS_FOR_FORWARD_SYNC=${2:-1}  # How many days back to sync to Wallabag

# --- Functions ---

check_connectivity() {
    echo "🔍 Checking connectivity..."
    
    # Check VPS
    if ! ssh -o ConnectTimeout=10 "$REMOTE_USER@$REMOTE_HOST" "echo 'VPS reachable'" > /dev/null 2>&1; then
        echo "❌ VPS not reachable"
        return 1
    fi
    echo "✅ VPS reachable"
    
    # Check NAS/Wallabag (optional, might be on local network only)
    if curl -s --connect-timeout 5 --max-time 10 "$WALLABAG_URL/api/info" > /dev/null 2>&1; then
        echo "✅ NAS/Wallabag reachable"
        WALLABAG_REACHABLE=true
    else
        echo "⚠️  NAS/Wallabag not reachable (might be local network only)"
        WALLABAG_REACHABLE=false
    fi
    
    return 0
}

# Enhanced push function with Wallabag sync
push_to_server() {
    echo "🔄 Starting bidirectional sync (push phase)..."
    
    # Step 1: Traditional ArchiveBox sync
    echo "📤 Syncing ArchiveBox files to server..."
    rsync -avz --info=progress2 "$LOCAL_DIR/archive/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/archive/"
    rsync -avz --info=progress2 "$LOCAL_DIR/sources/" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/sources/"
    
    # Step 2: Rebuild server ArchiveBox index
    echo "🔧 Rebuilding server ArchiveBox index..."
    ssh -t "$REMOTE_USER@$REMOTE_HOST" "cd $REMOTE_DIR && $REMOTE_ARCHIVEBOX_BINARY init"
    
    # Step 3: Trigger reverse sync on server (Wallabag → ArchiveBox)
    echo "🔄 Triggering reverse sync on server (Wallabag → ArchiveBox)..."
    ssh "$REMOTE_USER@$REMOTE_HOST" "bash $REVERSE_SYNC_SCRIPT" 2>/dev/null || echo "ℹ️  Reverse sync not available or failed"
    
    # Step 4: Forward sync (ArchiveBox → Wallabag) if reachable
    if [ "$WALLABAG_REACHABLE" = true ]; then
        echo "📲 Syncing recent ArchiveBox content to Wallabag..."
        sync_to_wallabag_forward
    fi
    
    echo "✅ Bidirectional push complete"
}

# Enhanced pull function
pull_from_server() {
    echo "🔄 Starting bidirectional sync (pull phase)..."
    
    # Step 1: Trigger reverse sync on server first
    echo "🔄 Triggering reverse sync on server (Wallabag → ArchiveBox)..."
    ssh "$REMOTE_USER@$REMOTE_HOST" "bash $REVERSE_SYNC_SCRIPT" 2>/dev/null || echo "ℹ️  Reverse sync not available or failed"
    
    # Step 2: Traditional ArchiveBox pull
    echo "📥 Pulling ArchiveBox files from server..."
    rsync -avz --info=progress2 "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/archive/" "$LOCAL_DIR/archive/"
    rsync -avz --info=progress2 "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/sources/" "$LOCAL_DIR/sources/"
    
    # Step 3: Rebuild local index
    echo "🔧 Rebuilding local ArchiveBox index..."
    cd "$LOCAL_DIR" && archivebox init
    
    # Step 4: Forward sync if Wallabag reachable
    if [ "$WALLABAG_REACHABLE" = true ]; then
        echo "📲 Syncing recent content to Wallabag..."
        sync_to_wallabag_forward
    fi
    
    echo "✅ Bidirectional pull complete"
}

sync_to_wallabag_forward() {
    echo "📤 Forward sync: ArchiveBox → Wallabag (last $DAYS_FOR_FORWARD_SYNC days)"
    
    # Get Wallabag token
    TOKEN=$(get_wallabag_token)
    if [ -z "$TOKEN" ]; then
        echo "❌ Could not authenticate with Wallabag"
        return 1
    fi
    
    # Find recent ArchiveBox entries
    CUTOFF_TIMESTAMP=$(date +%s -d "$DAYS_FOR_FORWARD_SYNC days ago")
    SYNCED_COUNT=0
    
    find "$LOCAL_DIR/archive/" -maxdepth 1 -mindepth 1 -type d | while read -r SNAPSHOT_DIR; do
        SNAPSHOT_TIMESTAMP=$(basename "$SNAPSHOT_DIR")
        SNAPSHOT_INT=${SNAPSHOT_TIMESTAMP%.*}
        
        if (( SNAPSHOT_INT > CUTOFF_TIMESTAMP )); then
            INDEX_FILE="$SNAPSHOT_DIR/index.json"
            
            if [ -f "$INDEX_FILE" ]; then
                URL=$(python3 -c "import sys, json; print(json.load(open(sys.argv[1]))['url'])" "$INDEX_FILE" 2>/dev/null)
                TITLE=$(python3 -c "import sys, json; print(json.load(open(sys.argv[1]))['title'])" "$INDEX_FILE" 2>/dev/null)
                
                if [ -n "$URL" ] && [ -n "$TITLE" ]; then
                    echo "📤 Sending to Wallabag: $TITLE"
                    send_to_wallabag "$URL" "$TITLE" "$TOKEN"
                    ((SYNCED_COUNT++))
                fi
            fi
        fi
    done
    
    echo "✅ Sent $SYNCED_COUNT articles to Wallabag"
}

get_wallabag_token() {
    curl -s -X POST "$WALLABAG_URL/oauth/v2/token" \
        -H "Content-Type: application/json" \
        -d "{
            \"grant_type\": \"password\",
            \"client_id\": \"$WALLABAG_CLIENT_ID\",
            \"client_secret\": \"$WALLABAG_CLIENT_SECRET\",
            \"username\": \"$WALLABAG_USERNAME\",
            \"password\": \"$WALLABAG_PASSWORD\"
        }" | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])" 2>/dev/null
}

send_to_wallabag() {
    local url="$1"
    local title="$2"
    local token="$3"
    
    echo "📤 Sending to Wallabag: $title"
    
    # Send to Wallabag
    curl -s -X POST "$WALLABAG_URL/api/entries.json" \
        -H "Authorization: Bearer $token" \
        -H "Content-Type: application/json" \
        -d "{\"url\": \"$url\", \"tags\": \"archivebox-sync\"}" > /dev/null 2>&1
    
    # Wait a moment for processing
    sleep 3
    
    # Check if the article was actually created by searching recent entries
    local check_response=$(curl -s -H "Authorization: Bearer $token" \
        "$WALLABAG_URL/api/entries.json?perPage=20&order=desc&sort=created" 2>/dev/null || echo "")
    
    # Look for the URL in recent entries (escape special regex characters)
    local escaped_url=$(echo "$url" | sed 's/[[\.*^$()+?{|]/\\&/g')
    if echo "$check_response" | grep -q "$escaped_url"; then
        echo "✅ Confirmed: Article successfully added to Wallabag"
        return 0
    else
        echo "❌ Failed: Article not found in Wallabag after submission"
        return 1
    fi
}

# --- Main Script Logic ---

if ! check_connectivity; then
    echo "❌ Connectivity check failed"
    exit 1
fi

case "$1" in
    "push")
        push_to_server
        ;;
    "pull")
        pull_from_server
        ;;
    *)
        echo "Usage: $0 [push|pull] [days_for_wallabag_sync]"
        echo ""
        echo "Commands:"
        echo "  push   - Sync local changes to server + bidirectional sync"
        echo "  pull   - Sync server changes to local + bidirectional sync"
        echo ""
        echo "Examples:"
        echo "  $0 push     # Sync last 1 day to Wallabag"
        echo "  $0 pull 3   # Pull from server, sync last 3 days to Wallabag"
        exit 1
        ;;
esac
EOF

# Make executable
chmod +x ~/sync_archivebox_bidirectional.sh

# IMPORTANT: Edit the script to update all the configuration variables
# Replace all placeholder values with your actual settings
```

## Part 4: Mobile Apps Setup

### 4.1 iOS/Android Wallabag App

1. **Download** the Wallabag app from App Store/Play Store
2. **Configure server settings**:
   - **Server URL**: `http://YOUR_NAS_HOSTNAME:8082` (on home network) or your external IP
   - **Username**: `your_admin_username`
   - **Password**: `your_secure_password`
3. **Test** by saving an article from your mobile browser

### 4.2 Browser Integration

Most mobile browsers will show "Share to Wallabag" option once the app is installed.

## Part 5: Daily Usage Workflow

### 5.1 Morning Sync (Pull from server)

```bash
# Pull any URLs saved from mobile overnight + sync recent to Wallabag
~/sync_archivebox_bidirectional.sh pull
```

### 5.2 Evening Sync (Push to server)

```bash
# Push your local additions + sync recent content to Wallabag  
~/sync_archivebox_bidirectional.sh push
```

### 5.3 Mobile → Archive Flow

1. **On phone**: Save article URL to Wallabag using mobile app
2. **VPS automatically** (every 30 min): Pulls new URLs from Wallabag → Archives them
3. **Next sync**: Your local machine gets the newly archived content
4. **Reading**: Article appears in both Wallabag (for reading) and ArchiveBox (for archival)

## Part 6: Advanced Features

### 6.1 EPUB Digest Generation (Optional)

Add automatic daily digest creation to your VPS:

```bash
# Create digest script on VPS (requires pandoc installation)
cat > /home/your_vps_user/create_digest_epub.sh << 'EOF'
#!/bin/bash

ARCHIVEBOX_DIR="/path/to/your/archivebox/data"
EXPORT_DIR="/home/your_vps_user/epub_exports"
DAYS_AGO=${1:-1}

echo "▶️  Starting digest creation for the last $DAYS_AGO day(s)..."
mkdir -p "$EXPORT_DIR"

WORK_DIR=$(mktemp -d)
trap 'rm -rf "$WORK_DIR"' EXIT

COVER_FILE="$WORK_DIR/00_cover.html"
ARTICLE_COUNTER=0
ARTICLE_LINKS=""
declare -a ARTICLE_FILES

CUTOFF_TIMESTAMP=$(date +%s -d "$DAYS_AGO days ago")

while IFS= read -r -d $'\0' SNAPSHOT_DIR; do
    SNAPSHOT_TIMESTAMP=$(basename "$SNAPSHOT_DIR")
    SNAPSHOT_INT=${SNAPSHOT_TIMESTAMP%.*}
    SOURCE_HTML="$SNAPSHOT_DIR/readability/content.html"

    if (( SNAPSHOT_INT > CUTOFF_TIMESTAMP )); then
        if [ -f "$SOURCE_HTML" ]; then
            ((ARTICLE_COUNTER++))
            FILENAME=$(printf "%02d" $ARTICLE_COUNTER).html
            FILEPATH="$WORK_DIR/$FILENAME"
            cp "$SOURCE_HTML" "$FILEPATH"
            
            TITLE=$(python3 -c "import sys, json; print(json.load(open(sys.argv[1]))['title'])" "$SNAPSHOT_DIR/index.json" 2>/dev/null || echo "Untitled Article")
            
            TEMP_FILE="$WORK_DIR/temp.html"
            echo "<h1>$TITLE</h1>" > "$TEMP_FILE"
            cat "$FILEPATH" >> "$TEMP_FILE"
            mv "$TEMP_FILE" "$FILEPATH"
            
            ARTICLE_LINKS+="<li><a href=\"$FILENAME\">$TITLE</a></li>\n"
            ARTICLE_FILES+=("$FILEPATH")
        fi
    fi
done < <(find "$ARCHIVEBOX_DIR/archive/" -maxdepth 1 -mindepth 1 -type d -print0)

if [ "$ARTICLE_COUNTER" -eq 0 ]; then
    echo "✅ No new articles found for the digest."
    exit 0
fi

DIGEST_DATE=$(date +"%A, %B %d, %Y")
FINAL_EPUB_FILENAME="Digest_$(date +%Y-%m-%d)_${DAYS_AGO}days.epub"
FINAL_EPUB_PATH="$EXPORT_DIR/$FINAL_EPUB_FILENAME"

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
EOF

chmod +x /home/your_vps_user/create_digest_epub.sh

# Add to cron for daily 5 AM generation
(crontab -l 2>/dev/null; echo '0 5 * * * /home/your_vps_user/create_digest_epub.sh > /tmp/digest_creation.log 2>&1') | crontab -
```

### 6.2 Local EPUB Download Script

```bash
# Create script to download EPUBs to local machine
cat > ~/get_epubs.sh << 'EOF'
#!/bin/bash

LOCAL_EPUB_DIR="$HOME/Downloads/EPUBS_from_ArchiveBox"
REMOTE_USER="your_vps_user"
REMOTE_HOST="YOUR_VPS_IP"
REMOTE_EPUB_DIR="/home/your_vps_user/epub_exports"

# Auto-detect the best rsync command on macOS
RSYNC_CMD="rsync"
PROGRESS_FLAG="-v"

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

"$RSYNC_CMD" -az "$PROGRESS_FLAG" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_EPUB_DIR/" "$LOCAL_EPUB_DIR/"

echo "✅ Sync complete. New EPUBs are in $LOCAL_EPUB_DIR"
open "$LOCAL_EPUB_DIR"
EOF

chmod +x ~/get_epubs.sh
```

## Part 7: Troubleshooting

### 7.1 Common Issues

**Wallabag API returns 500 errors but articles save**: This is normal behavior due to content extraction issues. Our scripts handle this correctly by verifying article creation rather than relying on HTTP status codes.

**VPS can't reach NAS**: The reverse sync will skip gracefully and try again later when connectivity is restored.

**Permission issues**: 
```bash
# On NAS
sudo chown -R 1000:1000 /path/to/your/storage/wallabag/

# On VPS
chown -R your_vps_user:your_vps_user /path/to/your/archivebox/data
```

**SSH connectivity issues**:
```bash
# Test SSH to VPS
ssh -o ConnectTimeout=10 your_vps_user@YOUR_VPS_IP "echo 'Connection test'"

# Regenerate SSH keys if needed
ssh-keygen -t rsa -b 4096
ssh-copy-id your_vps_user@YOUR_VPS_IP
```

**ArchiveBox database corruption**:
```bash
# Rebuild index safely
cd /path/to/archivebox/data && archivebox init
```

### 7.2 Monitoring and Logs

**VPS reverse sync logs**:
```bash
# Live monitoring
tail -f /home/your_vps_user/reverse_sync.log

# Check cron execution
tail -f /tmp/reverse_sync_cron.log
```

**NAS container logs**:
```bash
# Wallabag application logs
sudo docker logs wallabag --tail 50

# Database logs
sudo docker logs wallabag-db --tail 20
```

**Local sync testing**:
```bash
# Test connectivity before sync
~/sync_archivebox_bidirectional.sh

# Verbose rsync for debugging
rsync -avz --info=progress2 --dry-run ~/path/to/archivebox/data/archive/ your_vps_user@YOUR_VPS_IP:/path/to/archivebox/data/archive/
```

### 7.3 Performance Optimization

**Reduce sync frequency for large archives**:
```bash
# Edit cron job to run hourly instead of every 30 minutes
crontab -e
# Change: */30 * * * * to 0 * * * *
```

**Exclude large files from sync**:
```bash
# Add to rsync commands in sync script:
# --exclude="*.mp4" --exclude="*.zip" --exclude="*.tar.gz"
```

**Wallabag performance tuning**:
```bash
# Inside wallabag container
sudo docker exec wallabag php bin/console cache:clear --env=prod --no-warmup
sudo docker exec wallabag php bin/console cache:warmup --env=prod
```

## Part 8: Security and Network Configuration

### 8.1 External Access Configuration

For accessing Wallabag from outside your home network:

**Option 1: Port Forwarding**
```bash
# Configure your router to forward port 8082 to YOUR_NAS_HOSTNAME:8082
# Then use your external IP in mobile apps
```

**Option 2: Reverse Proxy with SSL** (Recommended)
```bash
# On NAS, set up nginx proxy with Let's Encrypt
# This provides HTTPS access for secure external connectivity
```

**Option 3: VPN Access**
```bash
# Use your existing VPN to access home network
# No external ports needed, most secure option
```

### 8.2 Security Best Practices

- **Firewall rules**: Only expose necessary ports
- **VPN access**: Prefer VPN over port forwarding for external access
- **Regular updates**: Keep all containers and systems updated
- **Strong passwords**: Use unique, strong passwords for all services
- **API credential rotation**: Periodically update Wallabag API credentials

### 8.3 Data Privacy

- **Data location**: All data stays within your controlled infrastructure
- **No cloud dependencies**: Fully self-hosted solution
- **Traffic encryption**: Use HTTPS/SSL for external access
- **Local processing**: All content extraction and archiving happens locally

## Part 9: Backup and Recovery

### 9.1 Complete System Backup

**NAS Wallabag Data**:
```bash
# Backup Wallabag data and database
sudo docker exec wallabag-db pg_dump -U wallabag wallabag > wallabag_backup_$(date +%Y%m%d).sql
tar -czf wallabag_data_$(date +%Y%m%d).tar.gz /path/to/your/storage/wallabag/
```

**VPS ArchiveBox Data**:
```bash
# On VPS - backup entire ArchiveBox directory
cd /home/your_vps_user
tar -czf archivebox_backup_$(date +%Y%m%d).tar.gz path/to/archivebox/data/
```

**Local ArchiveBox Data**:
```bash
# On local machine - backup ArchiveBox
cd ~/path/to/your/local/archivebox
tar -czf archivebox_local_backup_$(date +%Y%m%d).tar.gz data/
```

### 9.2 Recovery Procedures

**Restore Wallabag**:
```bash
# Stop containers
sudo docker stop wallabag wallabag-db

# Restore database
sudo docker start wallabag-db
sleep 10
cat wallabag_backup_YYYYMMDD.sql | sudo docker exec -i wallabag-db psql -U wallabag wallabag

# Restore data files
sudo tar -xzf wallabag_data_YYYYMMDD.tar.gz -C /

# Restart
sudo docker start wallabag
```

**Restore ArchiveBox**:
```bash
# Extract backup
tar -xzf archivebox_backup_YYYYMMDD.tar.gz

# Rebuild index
cd restored_directory && archivebox init
```

## Part 10: Maintenance Schedule

### 10.1 Daily (Automated)
- ✅ Reverse sync every 30 minutes (VPS → Wallabag → ArchiveBox)
- ✅ EPUB digest generation at 5 AM (if configured)
- ✅ Container health checks

### 10.2 Weekly (Manual)
- 🔄 Run bidirectional sync: `~/sync_archivebox_bidirectional.sh pull`
- 📊 Check disk space usage on all systems
- 📋 Review sync logs for any errors

### 10.3 Monthly (Manual)
- 💾 Create full system backups
- 🔒 Review and update passwords/API keys
- 🧹 Clean up old EPUB digests and logs
- ⬆️ Update Docker containers

### 10.4 Quarterly (Manual)
- 🔍 Security audit of exposed services
- 📈 Performance review and optimization
- 🗂️ Archive organization and cleanup

## Part 11: Configuration Checklist

Before deploying, ensure you have updated all placeholder values:

### 11.1 NAS Configuration
- [ ] `YOUR_NAS_HOSTNAME` - Your NAS hostname or IP
- [ ] `/path/to/your/storage/` - Your NAS storage paths
- [ ] `your_secure_db_password` - PostgreSQL password
- [ ] `your_admin_username` - Wallabag admin username
- [ ] `your_secure_password` - Wallabag admin password
- [ ] `your_email@domain.com` - Your email address

### 11.2 VPS Configuration
- [ ] `YOUR_VPS_IP` - Your VPS IP address
- [ ] `your_vps_user` - Your VPS username
- [ ] `/path/to/your/archivebox/data` - ArchiveBox data directory
- [ ] `/path/to/archivebox/binary` - ArchiveBox binary path
- [ ] Wallabag credentials in reverse sync script

### 11.3 Local Machine Configuration
- [ ] `$HOME/path/to/your/local/archivebox/data` - Local ArchiveBox path
- [ ] All VPS connection details
- [ ] All Wallabag API credentials

### 11.4 API Credentials
- [ ] `your_client_id_here` - Wallabag API client ID
- [ ] `your_client_secret_here` - Wallabag API client secret

## Part 12: System Summary

This tutorial creates a comprehensive, self-hosted article archiving and reading system with the following architecture:

```
📱 Mobile Apps → 🏠 NAS Wallabag → ☁️ VPS ArchiveBox → 💻 Local ArchiveBox
     ↑                    ↑                    ↑                    ↑
  Save URLs         Reading UI          Master Archive      Offline Access
  Share Articles    Family Access       24/7 Availability   Full Control
  iOS/Android       Web Interface       Auto Sync           Local Backup
```

**Key Benefits**: 
- 📚 Complete article archiving with ArchiveBox
- 📖 Beautiful reading experience with Wallabag  
- 📱 Mobile integration for capturing content anywhere
- 🔄 Automatic bidirectional synchronization
- 📊 Optional daily EPUB digest generation
- 🏠 Self-hosted privacy and control
- ☁️ Cloud availability and backup
- 💻 Offline capabilities

**Network Scenarios Handled**:
- **Home network**: Full bidirectional sync works
- **Remote location**: ArchiveBox sync works, Wallabag sync skipped gracefully
- **Mobile only**: URLs saved to Wallabag are automatically archived when VPS reaches NAS
- **Offline work**: Both ArchiveBox instances work offline, sync when connectivity returns

The system provides true bidirectional synchronization, allowing you to save articles from mobile devices and have them automatically appear in your comprehensive ArchiveBox archive, while also making your locally archived content available for beautiful reading in Wallabag across all your devices.

## Getting Started

1. **Start with the NAS setup** (Part 1) to get Wallabag running
2. **Configure the VPS reverse sync** (Part 2) for mobile integration
3. **Set up local bidirectional sync** (Part 3) for full workflow
4. **Install mobile apps** (Part 4) for article capture
5. **Test the complete workflow** (Part 5) to ensure everything works
6. **Add optional features** (Parts 6+) as desired

Remember to update all placeholder values with your actual configuration before deploying!
