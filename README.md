# **Pterodactyl Migration - Manual**

Required:

> 2 VPS Open port (VPS Old and New)
> VPS Operating System is Ubuntu
> Terminal (PuTTY for Windows, Termux and Termius for Android)

Note:

> NAT VPS is not recommended
---

## **Migration Steps**

### **VPS Old:**

1. Backup Database:
    ```bash
    mysqldump -u root -p --all-databases > /alldb.sql
    ```

2. Backup Pterodactyl, SSL, nginx files and their configurations
   ```bash
   tar -cvpzf backup.tar.gz /etc/letsencrypt /var/www/pterodactyl /etc/nginx/sites-available/pterodactyl.conf /alldb.sql
   ```
3. Node Data Backup:
    ```bash
    tar -cvzf node.tar.gz /var/lib/pterodactyl /etc/pterodactyl
    ```

### **Reconfigure domain IP in cloudflare**

1. Go to CloudFlare: https://dash.cloudflare.com
2. Select your domain
3. Go to DNS
4. Change your old IP to the New IP in your DNS Records
    
### **VPS New:**

1. Run Auto Installer Panel Same Node, Do Not Fill HTTPS and SSL Options:
    ```bash
    bash <(curl -s https://pterodactyl-installer.se)
    ```

2. Download Backup Panel and Node Files to New VPS:
    ```bash
    scp root@IP_OLD:/root/{backup.tar.gz,node.tar.gz} /
    ```
    
3. Extract Backup Panel File:
    ```bash
    tar -xvpzf /backup.tar.gz -C /
    ```

4. Restart Nginx:
    ```bash
    systemctl restart nginx
    ```
    
5. Extract Node Backup Files:
    ```bash
    tar -xvzf /node.tar.gz -C /
    ```

6. Restore Database
    ```bash
    mysql -u root -p < /alldb.sql
    ```

7. Update Database IP
    ```bash
    mysql
    ```
    ```mysql
    UPDATE allocations
    SET ip = 'IP New'
    WHERE ip = 'IP Old';
    ```
8. Restart Wings:
    ```bash
    systemctl restart wings
    ```
---

## Troubleshooting DNS Propagation Issues

DNS updates sometimes take time to propagate across the network, which can cause the network you are using to still detect the old IP. To solve this problem, you can do the following steps: flush DNS, clear Cloudflare cache (if any), and clear browser cache.

### 1. Flush DNS

To clear the DNS cache on your computer, run the following command in Command Prompt or terminal:

```bash
ipconfig /flushdns
```

### 2. Clear Browser Cache

To clear the cache on a browser, use the following shortcut:

```
CTRL + F5
```

### 3. Clear Cloudflare Cache

If you are using Cloudflare, there are two methods to clear the cache:

**Manual Method:**

- Entered into: **Domain > Caching > Caching Configuration > Purge Everything**

**API Method:**

To clear all cache using the Cloudflare API, use the following command:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache" \
     -H "Authorization: Bearer APIKEY" \
     -H "Content-Type: application/json" \
     --data '{"purge_everything":true}'
```

---