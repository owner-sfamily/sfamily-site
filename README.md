# SFamily — marketing site

A static landing page. No build step, no dependencies, no JavaScript: three
files and two images. Open `index.html` in a browser to see it exactly as it
will appear in production.

```
index.html
assets/css/styles.css
assets/img/sfamily-mark.png       96×96   — the lockup mark
assets/img/sfamily-mark-1024.png  1142×   — master, for future retina use
assets/img/app-icon-512.png       512×512 — favicon and og:image
```

## Why it looks the way it does

Every colour is copied verbatim from the app's single source of truth,
`famzone-flutter/lib/core/theme/app_colors.dart`, and the token block at the
top of `styles.css` names the Dart constant beside each one. **If a colour
changes in the app, change it here too.** A landing page that is nearly the
brand colour looks worse than one that is openly a different colour.

The typeface is **Poppins**, which is what the app bundles
(`assets/fonts/Poppins-SemiBold.ttf`). The page loads the full weight range
from Google Fonts because the app only ships the 600 weight. There is a real
fallback stack, so the page is still correct if Google Fonts is blocked.

The page is deliberately **dark only**. The app has no light mode, and a light
version of this page would be a different product.

## What the page is for

Apple requires a **Support URL** for every App Store listing, and accepts a
**Marketing URL** as well. This page serves both:

- Marketing URL → `https://sfamily.co/`
- Support URL → `https://sfamily.co/#support`

The `#support` section names a real monitored address. Before submitting to
Apple, confirm that address actually receives mail — a Support URL that leads
to a dead mailbox is a rejection.

Legal pages are **not** duplicated here. They stay on the CMS at
`admin-cp.sfamily.co/p/...` so there is one copy of each, and the footer links
to them. Two copies of a privacy policy that disagree is exactly the
inconsistency a reviewer looks for.

## Deploying to the existing EC2 box

The API already runs on this machine at `admin-cp.sfamily.co` from
`/var/www/famzone`. This is a separate nginx server block on the same box —
nothing about the API changes.

### 1. Copy the files

```bash
ssh ubuntu@16.192.91.162
sudo mkdir -p /var/www/sfamily-site
sudo chown -R ubuntu:ubuntu /var/www/sfamily-site
```

Then from your machine:

```bash
scp -r ./* ubuntu@16.192.91.162:/var/www/sfamily-site/
```

Or, if you put this folder in its own git repo, clone it there and `git pull`
to deploy.

### 2. nginx server block

`/etc/nginx/sites-available/sfamily-site`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name sfamily.co www.sfamily.co;

    root /var/www/sfamily-site;
    index index.html;

    # A static site with no router: fall back to the page itself, never to a
    # directory listing.
    location / {
        try_files $uri $uri/ /index.html;
    }

    # The webroot is a git working copy, so .git/ sits inside it. Without
    # this, anyone can fetch /.git/config and walk the whole history —
    # which is how private repositories get downloaded from public sites.
    # Blocks every dotfile, not just .git.
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Not secret, just not worth serving.
    location = /README.md { deny all; }

    # Fingerprint-free assets, so keep the cache short enough that a CSS
    # change is visible the same day rather than next month.
    location ~* \.(css|js)$ {
        expires 1h;
        add_header Cache-Control "public";
    }

    location ~* \.(png|jpg|jpeg|svg|webp|ico|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public";
    }

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    gzip on;
    gzip_types text/css text/html image/svg+xml;
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/sfamily-site /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 3. DNS

Point `sfamily.co` and `www.sfamily.co` at the box with A records.

**Use the Elastic IP, not the current public IP.** If the instance restarts
without one, its IP changes — which breaks DNS here and, more urgently, breaks
the MSG91 whitelist that OTP login depends on.

### 4. HTTPS

```bash
sudo certbot --nginx -d sfamily.co -d www.sfamily.co
```

Certbot rewrites the server block above to listen on 443 and adds the redirect.
Apple will not accept an `http://` support URL.

## Before going live

- [ ] `ownerabhi@sfamily.co` receives mail
- [ ] The three footer links to `admin-cp.sfamily.co/p/...` all resolve
- [ ] HTTPS certificate issued, `http://` redirects to `https://`
- [ ] Open the page on a real phone — the hero artwork and the nav are the two
      places a narrow screen shows problems first
- [ ] Update `og:image` and `canonical` in `index.html` if the domain is not
      `sfamily.co`

## Things deliberately left out

**No analytics.** The privacy section states the app carries no third-party
analytics or tracking. Putting a tracker on the page that says so would be
the sort of contradiction people screenshot.

**No store badges.** The app is not on either store yet. A badge linking
nowhere, or implying availability that does not exist, is worse than the plain
line of text the hero carries now. Add the real badges when the listings go
live, and update the "Android in closed testing now · iOS coming soon" line in
the hero at the same time.

**No testimonials or download counts.** There are no users yet. Invented ones
would be fabrication, and the kind a reviewer can check.
