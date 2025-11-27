# The "Bulletproof" Wallabag Developer Setup Guide

**Goal:** Install Wallabag from source, configured for **Development** (code editing, debugging, unminified assets), avoiding the common pitfalls of PHP versions, Node incompatibilities, and missing database fixtures.

## 1\. Prerequisites (Strict Versioning)

Wallabag is picky. You must use these specific versions to avoid the errors we just fixed.

  * **PHP:** Version **8.2** or **8.3** (Do not use 8.4/8.5 yet).
  * **Node.js:** Version **20** (Required for the latest Webpack/Encore).
  * **Database:** PostgreSQL.
  * **System:** macOS (Homebrew) or Ubuntu/Debian.

### macOS Setup

```bash
# 1. Install specific PHP version
brew install php@8.3
brew link --overwrite --force php@8.3

# 2. Install Node 20
nvm install 20
nvm use 20

# 3. Install Postgres & Composer
brew install postgresql composer
brew services start postgresql
```

### Ubuntu/Debian Setup

```bash
# 1. Install PHP and required extensions (Vital!)
sudo apt update
sudo apt install php8.3 php8.3-cli php8.3-fpm php8.3-curl php8.3-gd php8.3-xml php8.3-mbstring php8.3-intl php8.3-zip php8.3-pgsql composer postgresql postgresql-contrib

# 2. Install Node 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

-----

## 2\. Clone and Install Backend

### Clone

```bash
git clone https://github.com/wallabag/wallabag.git
cd wallabag
```

### Install PHP Dependencies

**Important:** We are *not* using `--no-dev`. We want the debugging tools.

```bash
composer install
```

### Configure Database (`parameters.yml`)

During installation, you will be prompted. Use these **exact** settings to avoid the "Invalid Parameter" crash:

  * **database\_driver:** `pdo_pgsql`
  * **database\_host:** `127.0.0.1`
  * **database\_port:** `5432`
  * **database\_name:** `wallabag`
  * **database\_user:** `wallabag`
  * **database\_password:** `wallabag` (or your choice)
  * **database\_charset:** `utf8` (**CRITICAL:** Do NOT use the default `utf8mb4`)

*If you made a mistake, edit `app/config/parameters.yml` manually.*

-----

## 3\. Database Initialization (Manual Method)

We will skip `make install` (which is flaky) and build the database manually.

### Create Database & User

```bash
psql postgres -c "CREATE USER wallabag WITH PASSWORD 'wallabag';"
psql postgres -c "CREATE DATABASE wallabag OWNER wallabag;"
psql postgres -c "GRANT ALL PRIVILEGES ON DATABASE wallabag TO wallabag;"
```

### Create Schema

```bash
php -d memory_limit=512M bin/console doctrine:schema:create
```

### Create Admin User

```bash
php -d memory_limit=512M bin/console fos:user:create admin admin@example.com wallabag --super-admin
```

### ⚠️ Inject Missing Settings ( The "500 Error" Fix)

This step prevents the "Restricted Access" and "Matomo" crashes by manually populating the config tables.

```bash
psql -h 127.0.0.1 -U wallabag -d wallabag -c "
INSERT INTO wallabag_internal_setting (name, value, section) VALUES 
('restricted_access', '0', 'misc'),
('download_images_enabled', '1', 'entry'),
('default_mode', 'list', 'entry'),
('default_language', 'en', 'misc'),
('matomo_enabled', '0', 'misc'),
('matomo_host', 'localhost', 'misc'),
('matomo_site_id', '0', 'misc'),
('store_article_headers', '1', 'entry'),
('share_public', '1', 'entry'),
('import_with_rabbitmq', '0', 'import'),
('import_with_redis', '0', 'import');
"
```

```bash
# Also fix the user config bug
psql -h 127.0.0.1 -U wallabag -d wallabag -c "
INSERT INTO wallabag_config (id, user_id, items_per_page, language, action_mark_as_read, list_mode, display_thumbnails) 
SELECT 1, id, 12, 'en', 0, 0, 1 FROM wallabag_user WHERE username = 'admin'
ON CONFLICT DO NOTHING;"
```

-----

## 4\. Frontend Setup (Developer Mode)

We use `--ignore-scripts` to bypass the `husky` git-hook errors that often plague Node 20 installs.

```bash
# 1. Install dependencies
npm install --ignore-scripts

# 2. Build assets for DEVELOPMENT (Includes source maps, no minification)
npm run build:dev
```

-----

## 5\. Running the Application

### Clear Cache

Always clear the dev cache before starting.

```bash
php -d memory_limit=512M bin/console cache:clear
```

### Start the Server

We use the built-in PHP server. We serve the `web` folder.

```bash
php -S localhost:8000 -t web
```

### Accessing the App (Developer Mode)

Open your browser to:
👉 **[http://localhost:8000/app\_dev.php/](https://www.google.com/search?q=http://localhost:8000/app_dev.php/)**

  * **User:** `admin`
  * **Pass:** `wallabag`
  * **Verify:** You should see a **Debug Toolbar** (gray bar) at the bottom of the screen.

-----

## 6\. Developer Workflow Cheatsheet

Now that you are running, here is how to work on the code:

### Modifying PHP Code

1.  Edit files in `src/` or `app/`.
2.  Refresh browser (Changes are usually instant).
3.  If you change **Configuration** (`parameters.yml`, `config.yml`) or **Database Entities**:
    ```bash
    php -d memory_limit=512M bin/console cache:clear
    ```

### Modifying CSS / JavaScript

1.  Open a **second terminal**.
2.  Run the watcher:
    ```bash
    npm run watch
    ```
3.  Edit files in `assets/`.
4.  Webpack will automatically recompile them. Refresh browser to see changes.

### Switching to "Production" Mode (to test speed)

1.  Build prod assets: `npm run build:prod`
2.  Clear prod cache: `php -d memory_limit=512M bin/console cache:clear --env=prod`
3.  Visit: `http://localhost:8000/app.php/`

### Troubleshooting "500 Errors"

If you get a generic error page:

1.  Ensure you are using `app_dev.php` in the URL.
2.  Check the logs:
    ```bash
    tail -f var/logs/dev.log
    ```
