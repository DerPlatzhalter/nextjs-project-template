# 📱 WorkingFlow App - Deployment Guide

## 🚀 Wie Sie die App auf Ihr Handy und Server bringen

### 📋 Inhaltsverzeichnis
1. [Mobile App Installation](#mobile-app-installation)
2. [Server Setup](#server-setup)
3. [Produktions-Deployment](#produktions-deployment)
4. [Database Setup](#database-setup)
5. [Domain & SSL](#domain--ssl)
6. [Wartung & Updates](#wartung--updates)

---

## 📱 Mobile App Installation

### Option 1: Progressive Web App (PWA) - Empfohlen
Die WorkingFlow App ist bereits als PWA optimiert und kann direkt auf Ihr Handy installiert werden:

#### Android:
1. Öffnen Sie Chrome auf Ihrem Android-Gerät
2. Gehen Sie zu Ihrer App-URL (z.B. `https://ihre-domain.com`)
3. Tippen Sie auf das Menü (3 Punkte) → "Zum Startbildschirm hinzufügen"
4. Die App wird wie eine native App installiert

#### iPhone/iPad:
1. Öffnen Sie Safari auf Ihrem iOS-Gerät
2. Gehen Sie zu Ihrer App-URL
3. Tippen Sie auf das Teilen-Symbol → "Zum Home-Bildschirm"
4. Die App wird installiert und funktioniert offline

### Option 2: Native App mit Capacitor
Für eine echte native App können Sie Capacitor verwenden:

```bash
# Capacitor installieren
npm install @capacitor/core @capacitor/cli
npm install @capacitor/android @capacitor/ios

# Capacitor initialisieren
npx cap init WorkingFlow com.workingflow.app

# Plattformen hinzufügen
npx cap add android
npx cap add ios

# App builden und kopieren
npm run build
npx cap copy

# Native Apps öffnen
npx cap open android  # Für Android Studio
npx cap open ios      # Für Xcode
```

---

## 🖥️ Server Setup

### Option 1: VPS/Dedicated Server (Empfohlen)

#### 1. Server Vorbereitung (Ubuntu/Debian)
```bash
# System aktualisieren
sudo apt update && sudo apt upgrade -y

# Node.js installieren (Version 18+)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# PM2 für Prozess-Management
sudo npm install -g pm2

# Nginx für Reverse Proxy
sudo apt install nginx

# SSL mit Let's Encrypt
sudo apt install certbot python3-certbot-nginx
```

#### 2. App auf Server deployen
```bash
# Repository klonen oder Dateien hochladen
git clone https://github.com/ihr-username/workingflow.git
cd workingflow

# Dependencies installieren
npm install

# Production Build erstellen
npm run build

# PM2 Konfiguration erstellen
echo 'module.exports = {
  apps: [{
    name: "workingflow",
    script: "npm",
    args: "start",
    cwd: "/pfad/zu/ihrer/app",
    env: {
      NODE_ENV: "production",
      PORT: 3000
    }
  }]
}' > ecosystem.config.js

# App mit PM2 starten
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

#### 3. Nginx Konfiguration
```bash
# Nginx Konfiguration erstellen
sudo nano /etc/nginx/sites-available/workingflow

# Inhalt:
server {
    listen 80;
    server_name ihre-domain.com www.ihre-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

# Site aktivieren
sudo ln -s /etc/nginx/sites-available/workingflow /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

#### 4. SSL Zertifikat einrichten
```bash
# Let's Encrypt SSL
sudo certbot --nginx -d ihre-domain.com -d www.ihre-domain.com

# Auto-Renewal testen
sudo certbot renew --dry-run
```

### Option 2: Cloud Hosting Plattformen

#### Vercel (Einfachste Option)
```bash
# Vercel CLI installieren
npm install -g vercel

# In Ihrem Projekt-Ordner
vercel

# Folgen Sie den Anweisungen
# Ihre App ist automatisch unter https://ihr-projekt.vercel.app verfügbar
```

#### Netlify
```bash
# Netlify CLI installieren
npm install -g netlify-cli

# Build erstellen
npm run build

# Deployen
netlify deploy --prod --dir=.next
```

#### Railway
```bash
# Railway CLI installieren
npm install -g @railway/cli

# Einloggen und deployen
railway login
railway init
railway up
```

---

## 🗄️ Database Setup (Optional - für erweiterte Features)

### PostgreSQL Setup
```bash
# PostgreSQL installieren
sudo apt install postgresql postgresql-contrib

# Database erstellen
sudo -u postgres createdb workingflow
sudo -u postgres createuser workingflow_user

# Prisma für Database Management
npm install prisma @prisma/client
npx prisma init

# Schema definieren (prisma/schema.prisma)
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id          String   @id @default(cuid())
  username    String   @unique
  displayName String
  email       String   @unique
  bio         String?
  isAdmin     Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

# Migration ausführen
npx prisma migrate dev --name init
npx prisma generate
```

### Environment Variables
```bash
# .env.local erstellen
DATABASE_URL="postgresql://workingflow_user:password@localhost:5432/workingflow"
NEXTAUTH_SECRET="ihr-geheimer-schlüssel"
NEXTAUTH_URL="https://ihre-domain.com"
OPENROUTER_API_KEY="ihr-openrouter-key"
```

---

## 🌐 Domain & DNS Setup

### Domain konfigurieren
1. **Domain kaufen** (z.B. bei Namecheap, GoDaddy, Strato)
2. **DNS Records setzen:**
   ```
   A Record: @ → Ihre Server IP
   A Record: www → Ihre Server IP
   ```
3. **Warten** (DNS Propagation kann 24h dauern)

### Subdomain für API (Optional)
```
A Record: api → Ihre Server IP
CNAME: www.api → api.ihre-domain.com
```

---

## 🔧 Produktions-Optimierungen

### Performance Optimierungen
```javascript
// next.config.ts erweitern
const nextConfig = {
  // Kompression aktivieren
  compress: true,
  
  // Image Optimierung
  images: {
    domains: ['ihre-domain.com'],
    formats: ['image/webp', 'image/avif'],
  },
  
  // PWA Konfiguration
  experimental: {
    appDir: true,
  },
  
  // Security Headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'Referrer-Policy',
            value: 'origin-when-cross-origin',
          },
        ],
      },
    ]
  },
}
```

### PWA Manifest hinzufügen
```json
// public/manifest.json
{
  "name": "WorkingFlow",
  "short_name": "WorkingFlow",
  "description": "Professional Workspace Platform",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1e293b",
  "theme_color": "#3b82f6",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

---

## 🔒 Sicherheit & Backup

### Firewall Setup
```bash
# UFW Firewall konfigurieren
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### Automatische Backups
```bash
# Backup Script erstellen
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/workingflow"
APP_DIR="/pfad/zu/ihrer/app"

# Database Backup
pg_dump workingflow > $BACKUP_DIR/db_$DATE.sql

# Files Backup
tar -czf $BACKUP_DIR/files_$DATE.tar.gz $APP_DIR

# Alte Backups löschen (älter als 30 Tage)
find $BACKUP_DIR -name "*.sql" -mtime +30 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete

# Crontab für tägliche Backups
# 0 2 * * * /pfad/zum/backup-script.sh
```

---

## 📊 Monitoring & Logs

### PM2 Monitoring
```bash
# PM2 Status anzeigen
pm2 status

# Logs anzeigen
pm2 logs workingflow

# Monitoring Dashboard
pm2 monit

# Memory/CPU Usage
pm2 show workingflow
```

### Nginx Logs
```bash
# Access Logs
sudo tail -f /var/log/nginx/access.log

# Error Logs
sudo tail -f /var/log/nginx/error.log
```

---

## 🔄 Updates & Wartung

### App Updates
```bash
# Code aktualisieren
git pull origin main

# Dependencies aktualisieren
npm install

# Neu builden
npm run build

# App neu starten
pm2 restart workingflow
```

### System Updates
```bash
# System Updates
sudo apt update && sudo apt upgrade -y

# Node.js Updates
sudo npm install -g n
sudo n stable

# PM2 Updates
pm2 update
```

---

## 🆘 Troubleshooting

### Häufige Probleme

#### App startet nicht
```bash
# Logs prüfen
pm2 logs workingflow

# Port prüfen
sudo netstat -tulpn | grep :3000

# Prozesse prüfen
ps aux | grep node
```

#### SSL Probleme
```bash
# Zertifikat erneuern
sudo certbot renew

# Nginx Konfiguration testen
sudo nginx -t
```

#### Performance Probleme
```bash
# Memory Usage prüfen
free -h

# Disk Space prüfen
df -h

# PM2 Memory Monitoring
pm2 monit
```

---

## 📞 Support & Kontakt

Bei Problemen oder Fragen:
1. Prüfen Sie die Logs (`pm2 logs`)
2. Überprüfen Sie die Nginx Konfiguration
3. Testen Sie die Netzwerkverbindung
4. Kontaktieren Sie Ihren Hosting-Provider bei Server-Problemen

---

**🎉 Herzlichen Glückwunsch!** 
Ihre WorkingFlow App ist jetzt produktionsbereit und kann von überall auf der Welt genutzt werden!
