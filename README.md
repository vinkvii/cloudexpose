# cloudexpose
---

# VPN Reverse Proxy with Public Internet Exposure

This project allows you to expose your local application to the public internet securely using a VPN (WireGuard) through a VPS server. It uses a reverse proxy setup with `jc21/nginx-proxy-manager` for routing traffic, combined with `wg-easy` for easy VPN management.

## Features

- **Reverse Proxy**: Manage your HTTP and HTTPS traffic securely with `nginx-proxy-manager`.
- **VPN Server**: WireGuard VPN server using `wg-easy` to route traffic through your VPS.
- **Database**: MySQL container for managing data with `phpMyAdmin` for easy database administration.

## Prerequisites

- A **VPS** with a **public IP address** and **Docker** and **Docker Compose** installed.
- A domain name configured to point to your VPS (e.g., `vpn.yourdomain.com`).
- Basic knowledge of **DNS setup**: including configuring **A Records**, **CNAME records**, and other DNS settings needed to point your domain/subdomain to your VPS.
- Basic understanding of Docker, VPN clients, and networking.

## Setup

### 1. Domain Setup

- Configure your DNS settings with your domain registrar or DNS provider:
   - **A Record**: Point your domain or subdomain (e.g., `vpn.yourdomain.com`) to the public IP address of your VPS.
   - **CNAME Record**: If applicable, configure CNAME records to alias subdomains (e.g., `www.yourdomain.com`) to the main domain.

### 2. SSH into Your VPS

Log into your VPS using SSH. Replace `your-vps-ip` with the actual IP of your VPS.

```bash
ssh root@your-vps-ip
```

### 3. Clone this Repository on Your VPS

Once logged into your VPS, clone the repository and navigate into the directory:

```bash
git clone https://github.com/yourusername/vpn-reverse-proxy.git
cd vpn-reverse-proxy
```

### 4. Configure Environment Variables

Edit the `docker-compose.yml` file and configure the environment variables for your VPN. Change the `WG_HOST` environment variable under the `wg-easy` service to your VPS's public address or domain name.

```yaml
environment:
  - WG_HOST=vpn.yourdomain.com
```

### 5. Customize MySQL Credentials

Edit the MySQL credentials and database names in the `docker-compose.yml` file.

```yaml
environment:
  MYSQL_ROOT_PASSWORD: "rootPASS"
  MYSQL_DATABASE: "dbase"
  MYSQL_USER: "dbuser"
  MYSQL_PASSWORD: "dbpass"
```

### 6. Start the Docker Containers on Your VPS

Now you’re ready to start the services using Docker Compose. Run the following command on your VPS:

```bash
docker-compose up -d
```

This will download the necessary Docker images, create the containers, and start the services.

### 7. Create Client Config Using wg-easy

1. **Access wg-easy**:
   Visit `http://your-vps-ip:51821` or the domain you’ve configured (e.g., `http://vpn.yourdomain.com:51821`) to access the wg-easy management interface.

2. **Add a Client**:
   - In the wg-easy interface, click "Add Client".
   - Name the client (e.g., "local-machine").
   - Download the generated configuration file.

3. **Install WireGuard Client**:
   - Install a WireGuard client on your local machine. Clients are available for various platforms (Windows, macOS, Linux, iOS, Android).
   - Import the configuration file you downloaded from wg-easy into the WireGuard client.

4. **Connect to VPN**:
   - Enable the VPN connection in the WireGuard client. Your local machine is now connected to the VPN running on your VPS.

### 8. Configure Reverse Proxy in Nginx Proxy Manager

1. **Access Nginx Proxy Manager**:
   - Visit `http://your-vps-ip:81` or `http://vpn.yourdomain.com:81` to access the Nginx Proxy Manager.

2. **Create a Proxy Host Entry**:
   - In the Nginx Proxy Manager dashboard, go to "Proxy Hosts" and click "Add Proxy Host".
   - Set the domain/subdomain you want to route (e.g., `app.yourdomain.com`).
   - For the **Forward Hostname/IP**, enter the VPN IP of your local machine (e.g., `10.8.0.2`).
   - Set the **Forward Port** to the port where your application is running on your local machine.
   - Enable SSL and use Let's Encrypt to generate a certificate if needed.

3. **Finalize Proxy Settings**:
   - Save the proxy host entry. Now, any traffic to `app.yourdomain.com` will be routed through the VPN to your local machine, securely exposing your local application to the public internet.

## Docker Compose Configuration

```yaml
version: "3.8"
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: "vpn-reverse-proxy"
    restart: unless-stopped
    privileged: true
    network_mode: service:wg-easy
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "dbuser"
      DB_MYSQL_PASSWORD: "dbpass"
      DB_MYSQL_NAME: "dbase"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.disable_ipv6=0
    cap_add:
      - NET_ADMIN
      - SYS_MODULE

  db:
    image: "mysql"
    container_name: "mysql"
    volumes:
      - "./dbdata:/var/lib/mysql"
    command:
      - "--default-authentication-plugin=mysql_native_password"
    environment:
      MYSQL_ROOT_PASSWORD: "rootPASS"
      MYSQL_DATABASE: "dbase"
      MYSQL_USER: "dbuser"
      MYSQL_PASSWORD: "dbpass"

  phpmyadmin:
    image: "phpmyadmin/phpmyadmin"
    container_name: "pma"
    depends_on:
      - "db"
    environment:
      PMA_HOST: "db"
      PMA_PORT: "3306"
      UPLOAD_LIMIT: "256M"
  
  wg-easy:
    image: weejewel/wg-easy
    container_name: wg-easy
    volumes:
      - .:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
      - "80:80"
      - "81:81"
      - "443:443"
      - "82:80"
    environment:
      - WG_HOST=vpn.yourdomain.com
      - PASSWORD=foobar123
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

## Accessing Your Application

- Use your domain or VPS IP address to access the application exposed through Nginx Proxy Manager.
- Use the VPN credentials generated by `wg-easy` to connect your local machine to the VPN, ensuring that traffic is securely routed through the VPS.
- Configure reverse proxy entries in Nginx Proxy Manager to route traffic to your local machine using the VPN IP.

## Troubleshooting

- Ensure your VPS firewall allows traffic on the necessary ports (51820 for UDP, 80/443 for HTTP/HTTPS).
- Check the logs of the containers using `docker logs <container_name>` to identify any issues.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for more details.

---

This version includes detailed instructions on how to run the `docker-compose.yml` file on a VPS with a public IP and set up the VPN and reverse proxy using `wg-easy` and Nginx Proxy Manager.
