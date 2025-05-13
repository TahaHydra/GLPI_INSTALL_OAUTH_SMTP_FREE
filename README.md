# GLPI_INSTALL_OAUTH_SMTP_FREE
**Complete GLPI Setup on cPanel with Microsoft Azure OAuth and SMTP Integration**

---

### 🚀 1. GLPI Installation on cPanel Server

#### Environment Details:

* **OS**: Ubuntu 22.04
* **Server stack**: cPanel (WHM), Docker removed for this setup
* **Webserver**: Nginx (manual installation, Apache disabled)

#### 1.1 Required Packages:

```bash
sudo apt install -y php8.1 php8.1-cli php8.1-common php8.1-curl php8.1-gd php8.1-intl php8.1-mysql \
php8.1-ldap php8.1-xml php8.1-zip php8.1-mbstring php8.1-bcmath php8.1-soap \
php8.1-imap php8.1-apcu php8.1-bz2 php8.1-readline php8.1-fpm
```

* Added `ppa:ondrej/php` manually after removing `cpanel-exclude` apt rules.

#### 1.2 GLPI Version Used:

* `glpi-10.0.18`
* Extracted to: `/var/www/html/glpi`

---

### 🌐 2. NGINX Configuration

#### File: `/etc/nginx/sites-available/glpi`

```nginx
server {
    listen 80;
    server_name support.exco.fr;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name support.exco.fr;

    root /var/www/html/glpi/public;
    index index.php index.html;

    ssl_certificate     /etc/letsencrypt/live/support.exco.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/support.exco.fr/privkey.pem;

    access_log /var/log/nginx/glpi_access.log;
    error_log  /var/log/nginx/glpi_error.log;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php$ {
        include fastcgi_params;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }

    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy no-referrer-when-downgrade;
}
```

---

### 🌐 3. Let's Encrypt SSL

* Used `certbot` with standalone mode for HTTPS.
* Fixed issue where plugin and redirection were using `http` instead of `https`.
* Overrode plugin code to force `https` via `$_SERVER['HTTPS'] = 'on';` if needed.

---

### 👁 4. GLPI Single Sign-On (SSO)

#### Plugin Used:

* [glpi-singlesignon by edgardmessias](https://github.com/edgardmessias/glpi-singlesignon)

#### Fixes Applied:

* Fixed in configuration -> generale -> Change url to HTTPS
* Forced HTTPS return in `getBaseURL()` in `Toolbox.php`
* Used `generic` provider instead of `azure` to avoid errors.
* Enabled auto user creation in `findUser()`.
* Corrected domain check by unchecking "ne pas prendre en compte le domaine" in SSO settings.

---

### 🚀 5. Azure SSO App Registration

* Registered a new Azure App
* Used:

  * Client ID
  * Client Secret
  * Redirect URI: `https://support.exco.fr/plugins/singlesignon/front/callback.php/provider/1`

#### Permissions Required:

* `openid`, `profile`, `email`, `User.Read`
* Consent granted for tenant-wide access

---

### 📧 6. SMTP Configuration with OAuth (Microsoft 365)

#### Settings:

* **SMTP Host**: `smtp.office365.com`
* **Port**: `587`
* **Authentication**: OAuth
* **OAuth Provider**: Azure
* **Redirect URL**: `https://support.exco.fr/front/smtp_oauth2_callback.php`
* **Permissions Required**: `SMTP.Send`, `offline_access`, `email`, `openid`, `profile`

---

### 📊 7. Common Issues and Fixes

* **Problem**: GLPI redirected to HTTP → fixed by overriding `Toolbox.php` and enabling proper Nginx redirect.
* **Problem**: Emails not added to users → caused by "ne pas prendre en compte le domaine" being enabled.
* **Problem**: SMTP connection hijacked by cPanel (Exim server shown in `telnet`) →

  * Fixed by **disabling SMTP restrictions in WHM**:

    * WHM > Tweak Settings > Mail > Restrict outgoing SMTP → **Off**
* **Problem**: Debug mode left on → GLPI temporarily inaccessible. Fixed by editing config file or waiting session timeout.

---

### 🧰 8. Optional Notes (cPanel-specific)

* Nginx must be fully managed manually; **disable EasyApache Nginx proxying**.
* Remove Apache dependencies if not needed.
* Disable automatic SMTP redirection in cPanel.

---

### ✅ Final Checks

* ✅ SSO via Microsoft Azure works
* ✅ Auto user creation works
* ✅ GLPI uses HTTPS throughout
* ✅ SMTP sends mail using Microsoft 365 with OAuth2
* ✅ Email addresses are stored

---

### 🎓 Credits

Written by ChatGPT in collaboration with Taha Laachari during hands-on setup and debug.
