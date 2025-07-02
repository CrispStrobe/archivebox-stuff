# Complete Wallabag Installation Tutorial

A comprehensive guide to install and set up wallabag from source on macOS/Linux with PostgreSQL.

## Prerequisites

Before starting, ensure you have the following installed:

- **Git**: For cloning the repository
- **PHP 8.1+**: `brew install php` (macOS) or `sudo apt install php` (Ubuntu)
- **Composer**: PHP dependency manager
- **Node.js 18+**: Use nvm for version management
- **PostgreSQL**: Database server
- **Basic terminal knowledge**

## Step 1: Install Dependencies

### Install Composer (PHP Dependency Manager)

```bash
# macOS with Homebrew
brew install composer

# Ubuntu/Debian
sudo apt update
sudo apt install composer

# Manual installation (any OS)
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
sudo chmod +x /usr/local/bin/composer
```

### Install Node.js (for frontend assets)

```bash
# Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# Install and use Node.js 18
nvm install 18
nvm use 18
```

### Install PostgreSQL

```bash
# macOS
brew install postgresql
brew services start postgresql

# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

## Step 2: Clone and Install Wallabag

### Clone the Repository

```bash
git clone https://github.com/wallabag/wallabag.git
cd wallabag
```

### Install PHP Dependencies

```bash
composer install
```

### Configure Database Connection

During the composer install, you'll be prompted for database configuration. Use these settings:

```
database_driver: pdo_pgsql
database_host: 127.0.0.1
database_port: 5432
database_name: wallabag
database_user: wallabag
database_password: [your_secure_password]
domain_name: 'http://localhost:8000'
```

**Important**: Replace `[your_secure_password]` with a strong password of your choice.

## Step 3: Set Up PostgreSQL Database

### Create Database and User

```bash
# Connect to PostgreSQL
psql postgres

# In PostgreSQL shell, run these commands:
CREATE USER wallabag WITH PASSWORD '[your_secure_password]';
CREATE DATABASE wallabag OWNER wallabag;
GRANT ALL PRIVILEGES ON DATABASE wallabag TO wallabag;
\q
```

### Verify Database Configuration

Check that your `app/config/parameters.yml` file has the correct database settings:

```yaml
database_driver: pdo_pgsql
database_host: 127.0.0.1
database_port: 5432
database_name: wallabag
database_user: wallabag
database_password: '[your_secure_password]'
database_charset: utf8
```

## Step 4: Run Initial Installation

### Execute Installation Command

```bash
# Run the wallabag installer
make install
```

When prompted:
- **Reset database**: Choose `yes` for first installation
- **Create admin user**: Choose `no` (we'll create it manually later)

## Step 5: Create Admin User

### Create User with Proper Configuration

```bash
# Check if user exists and create config if missing
psql -h 127.0.0.1 -p 5432 -U wallabag -d wallabag -c "SELECT id, username, email, enabled FROM wallabag_user;"

# Create admin user
bin/console fos:user:create admin admin@example.com [your_admin_password] --super-admin --env=prod --quiet 2>/dev/null

# If the above fails due to missing config, create it manually:
psql -h 127.0.0.1 -p 5432 -U wallabag -d wallabag -c "
INSERT INTO wallabag_config (id, user_id, items_per_page, language, action_mark_as_read, list_mode, display_thumbnails) 
SELECT 1, id, 12, 'en', 0, 0, 1
FROM wallabag_user 
WHERE username = 'admin';"
```

**Security Note**: Replace `[your_admin_password]` with a strong password for your admin account.

## Step 6: Build Frontend Assets

### Install Node.js Dependencies

```bash
npm install
```

### Build Production Assets

```bash
npm run build:prod
```

## Step 7: Configure Asset Serving

### Update Domain Configuration

Edit `app/config/parameters.yml` to ensure the domain includes the port:

```yaml
domain_name: 'http://localhost:8000'
```

### Clear Cache

```bash
# Clear cache with increased memory limit
php -d memory_limit=512M bin/console cache:clear --env=prod --no-warmup
```

## Step 8: Start the Application

### Run the Development Server

```bash
bin/console server:run localhost:8000
```

### Access Wallabag

Open your browser and navigate to: `http://localhost:8000`

Log in with:
- **Username**: `admin`
- **Password**: `[your_admin_password]`

## Development Setup (Optional)

For development with automatic asset rebuilding:

### Terminal 1 - PHP Server
```bash
bin/console server:run localhost:8000
```

### Terminal 2 - Asset Watching
```bash
npm run watch
```

## Troubleshooting

### CSS Not Loading

If styling doesn't appear:

1. **Check asset URLs in browser developer tools**
2. **Verify domain_name in parameters.yml includes port 8000**
3. **Clear cache**: `php -d memory_limit=512M bin/console cache:clear --env=prod`
4. **Rebuild assets**: `npm run build:prod`

### Database Connection Issues

1. **Verify PostgreSQL is running**: `brew services list | grep postgresql` (macOS)
2. **Test database connection**: `psql -h 127.0.0.1 -p 5432 -U wallabag -d wallabag`
3. **Check parameters.yml database settings**

### Memory Issues During Cache Clear

If you encounter memory exhaustion:

```bash
# Increase PHP memory limit
php -d memory_limit=1G bin/console cache:clear --env=prod
```

### User Login Issues

If you get "getLanguage() on null" errors:

```bash
# Ensure user config exists
psql -h 127.0.0.1 -p 5432 -U wallabag -d wallabag -c "
SELECT * FROM wallabag_config WHERE user_id = (SELECT id FROM wallabag_user WHERE username = 'admin');"

# If no results, create config:
psql -h 127.0.0.1 -p 5432 -U wallabag -d wallabag -c "
INSERT INTO wallabag_config (id, user_id, items_per_page, language, action_mark_as_read, list_mode, display_thumbnails) 
SELECT 1, id, 12, 'en', 0, 0, 1
FROM wallabag_user 
WHERE username = 'admin';"
```

## Security Considerations

### Production Deployment

For production use:

1. **Use HTTPS**: Configure proper SSL certificates
2. **Strong passwords**: Use complex passwords for database and admin accounts
3. **Firewall**: Restrict database access to localhost only
4. **Updates**: Keep wallabag and dependencies updated
5. **Backups**: Regular database backups

### Environment Variables

Consider using environment variables for sensitive data:

```bash
export WALLABAG_DATABASE_PASSWORD="[your_secure_password]"
export WALLABAG_ADMIN_PASSWORD="[your_admin_password]"
```

## File Structure

After installation, your directory structure should look like:

```
wallabag/
├── app/
│   └── config/
│       └── parameters.yml
├── bin/
├── src/
├── web/
│   └── wallassets/
├── vendor/
├── node_modules/
├── composer.json
├── package.json
└── webpack.config.js
```

## Useful Commands

### User Management
```bash
# Create user
bin/console fos:user:create username email@example.com password

# Promote user to admin
bin/console fos:user:promote username ROLE_SUPER_ADMIN

# List users
bin/console fos:user:list
```

### Asset Management
```bash
# Build for production
npm run build:prod

# Build for development
npm run build:dev

# Watch for changes
npm run watch
```

### Database Management
```bash
# Update database schema
bin/console doctrine:schema:update --force

# Create database backup
pg_dump -h 127.0.0.1 -U wallabag wallabag > wallabag_backup.sql

# Restore database
psql -h 127.0.0.1 -U wallabag wallabag < wallabag_backup.sql
```

## Next Steps

After successful installation:

1. **Configure reading settings** in the admin panel
2. **Install browser extensions** for easy article saving
3. **Set up mobile apps** for reading on the go
4. **Configure import tools** to migrate from other services
5. **Explore API capabilities** for custom integrations

## Support

- **Official Documentation**: https://doc.wallabag.org
- **GitHub Issues**: https://github.com/wallabag/wallabag/issues
- **Community Support**: Matrix chat and forums

---

**Congratulations!** You now have a fully functional wallabag installation ready for saving and organizing web articles.
