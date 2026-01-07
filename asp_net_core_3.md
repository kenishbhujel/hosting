# ASP.NET Core 3.1 Hosting & SQL Server (Docker) Deployment Guide

This document explains **step-by-step** how to host an **ASP.NET Core 3.1 project on Ubuntu 22.04** and how to **restore a Microsoft SQL Server database using Docker**.

---

## ⚠️ Important Notes (Read First)

- **.NET Core 3.1 is EOL**, but still works on Ubuntu **22.04**
- **Ubuntu 24.04 does NOT support SQL Server packages properly**
- For database, we use **Docker + SQL Server 2019**
- Application uses **.NET Runtime only (SDK not required)**

---

# PART 1: HOSTING ASP.NET CORE 3.1 APPLICATION

## STEP 0: Install .NET Core 3.1 Runtime

```bash
sudo apt update
sudo apt install -y dotnet-runtime-3.1 aspnetcore-runtime-3.1
```

Verify:
```bash
dotnet --list-runtimes
```

---

## STEP 1: Create Application Folder & Set Permissions

```bash
sudo mkdir -p /var/www/counterbilling
sudo chown -R root:root /var/www/counterbilling
sudo chmod -R 755 /var/www/counterbilling
```

---

## STEP 2: Transfer Published Files (From Windows to Server)

On **Windows PowerShell**:

```powershell
cd C:\Users\sagar\Downloads\counterbilling
scp -r * root@69.164.245.198:/var/www/counterbilling
```

---

## STEP 3: Create systemd Service (Auto Start on Boot)

```bash
sudo nano /etc/systemd/system/counterbilling.service
```

Paste:

```ini
[Unit]
Description=CounterBilling .NET Core 3.1 App
After=network.target

[Service]
WorkingDirectory=/var/www/counterbilling
ExecStart=/usr/bin/dotnet /var/www/counterbilling/FiboCounterSystem.dll
Restart=always
RestartSec=10
SyslogIdentifier=counterbilling
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000

[Install]
WantedBy=multi-user.target
```

Save & exit (`CTRL+X → Y → ENTER`)

---

## STEP 4: Enable & Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable counterbilling
sudo systemctl start counterbilling
sudo systemctl status counterbilling
```

---

# PART 2: DATABASE SETUP (SQL SERVER USING DOCKER)

## Why Docker?

- SQL Server **does not run natively on Ubuntu 24.04**
- Docker provides **stable SQL Server 2019**
- Supports **.bak restore perfectly**

---

## STEP 1: Install Docker (If Not Installed)

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

---

## STEP 2: Run SQL Server 2019 Container

```bash
sudo docker run -e 'ACCEPT_EULA=Y' \
-e 'SA_PASSWORD=StrongPass123!' \
-p 1433:1433 \
--name sqlserver \
-d mcr.microsoft.com/mssql/server:2019-latest
```

Verify:
```bash
sudo docker ps
```

---

## STEP 3: Copy .bak File (Host → Container)

```bash
sudo docker cp pathibhara.bak sqlserver:/var/opt/mssql/backup/pathibhara.bak
```

---

## STEP 4: Verify Backup File Exists

```bash
sudo docker exec -it sqlserver ls -lh /var/opt/mssql/backup
```

---

## STEP 5: Check Logical File Names (.bak Analysis)

```bash
sqlcmd -S 127.0.0.1,1433 -U SA -P 'StrongPass123!'
```

```sql
RESTORE FILELISTONLY
FROM DISK = '/var/opt/mssql/backup/pathibhara.bak';
GO
```

Note **LogicalName** values carefully.

---

## STEP 6: Restore Database

```sql
RESTORE DATABASE pathibhara
FROM DISK = '/var/opt/mssql/backup/pathibhara.bak'
WITH
MOVE 'restrodb' TO '/var/opt/mssql/data/pathibhara.mdf',
MOVE 'restrodb_log' TO '/var/opt/mssql/data/pathibhara_log.ldf',
REPLACE;
GO
```

---

## STEP 7: Verify Database

```sql
SELECT name FROM sys.databases;
GO
```

---

# PART 3: CONNECTION STRING (ASP.NET CORE)

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=127.0.0.1,1433;Database=pathibhara;User Id=SA;Password=StrongPass123!;TrustServerCertificate=True;"
}
```

---

# PART 4: FINAL CHECKS

Restart services:
```bash
sudo systemctl restart counterbilling
```

Check logs if needed:
```bash
journalctl -u counterbilling -n 50 --no-pager
```

---

## ✅ Deployment Completed Successfully

You now have:
- ASP.NET Core 3.1 app running on Ubuntu 22.04
- SQL Server 2019 running in Docker
- Database restored from .bak
- App auto-starting on reboot

---

### 🔐 Recommended Next Steps
- Setup **Nginx reverse proxy**
- Enable **HTTPS (Let's Encrypt)**
- Disable SQL `SA` login after production setup
- Migrate project to **.NET 6+** when possible

