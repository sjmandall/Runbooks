##Runbook: Configure a Self-Signed TLS Certificate for Nginx in Docker
#Task
Set up HTTPS for a PHP application running behind Nginx in Docker by creating a self-signed TLS certificate, configuring Nginx to use it, and validating that the site loads over HTTPS locally.

Environment
Host OS: Ubuntu on WSL, with Windows host entries used for local name resolution.
​

Web server: Nginx running in a Docker container.
​

App stack: Nginx + PHP-FPM + MariaDB in Docker Compose.
​

Local hostname used: webpage.

Objective
Access the site as http://webpage instead of http://localhost.

Add HTTPS support with a self-signed certificate.

Understand why the browser shows a warning for a self-signed certificate and how to validate that TLS is still working correctly.

Initial Nginx Configuration Found
The existing Nginx config file was located at:

bash
/home/adminOps/dockerstackphp/php-docker-task/nginx/default.conf
The original configuration used port 80 and served PHP from /var/www/html:

text
server {
    listen 80;
    server_name webpage localhost;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
This confirmed that the document root inside the container was /var/www/html and PHP requests were being forwarded to the php service on port 9000.

Steps Performed
1. Check the existing site configuration
bash
cd /home/adminOps/dockerstackphp/php-docker-task/nginx
cat default.conf
Purpose:

Confirm the current server_name.

Identify the root path.

Verify how PHP was being passed to the PHP-FPM container.

2. Create a directory for TLS certificate files
From the project root:

bash
cd /home/adminOps/dockerstackphp/php-docker-task
mkdir -p nginx/ssl
Purpose:

Store the self-signed certificate and private key in a predictable location on the host.

Mount this directory into the Nginx container later.

3. Generate a self-signed certificate
Command used:

bash
openssl req -x509 -newkey rsa:2048 \
  -keyout nginx/ssl/mysite.key \
  -out nginx/ssl/mysite.crt \
  -days 1 \
  -nodes \
  -subj "/C=IN/ST=UP/L=Noida/O=Training/CN=webpage"
Explanation:

-x509 creates a self-signed certificate directly.

-newkey rsa:2048 creates a new 2048-bit RSA private key.
​

-keyout writes the private key to nginx/ssl/mysite.key.

-out writes the certificate to nginx/ssl/mysite.crt.

-days 1 makes the certificate valid for one day.

-nodes keeps the key unencrypted so Nginx can read it without a passphrase.

CN=webpage matches the local hostname used in server_name.

4. Verify certificate validity
bash
openssl x509 -in nginx/ssl/mysite.crt -noout -dates
Purpose:

Check notBefore and notAfter.

Confirm the certificate is valid and not expired.
​

5. Update Nginx configuration for HTTP and HTTPS
The default.conf file was updated to include:

An HTTP block on port 80 that redirects traffic to HTTPS.

An HTTPS block on port 443 using the self-signed certificate.

Final configuration:

text
server {
    listen 80;
    server_name webpage localhost;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name webpage localhost;

    root /var/www/html;
    index index.php index.html;

    ssl_certificate /etc/nginx/ssl/mysite.crt;
    ssl_certificate_key /etc/nginx/ssl/mysite.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
6. Ensure Docker exposes port 443 and mounts the certificate path
The Nginx service in docker-compose.yml needed to expose both HTTP and HTTPS and mount the certificate directory into the container.

Required structure:

text
services:
  nginx:
    image: nginx:alpine
    container_name: nginx_web
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/ssl:/etc/nginx/ssl
      - ./:/var/www/html
    depends_on:
      - php
Purpose:

443:443 publishes HTTPS to the host.

./nginx/ssl:/etc/nginx/ssl makes the certificate available where Nginx expects it.
​

./:/var/www/html matches the document root configured in default.conf.

7. Restart containers
bash
docker compose down
docker compose up -d
docker ps
Purpose:

Apply the updated Nginx config and Docker mappings.

Confirm that the Nginx container is running and that both ports 80 and 443 are published.

8. Validate HTTPS with curl
bash
curl -vk https://webpage/
Observed result:

TLS handshake completed successfully.

Nginx returned HTTP/1.1 200 OK.

The certificate subject showed CN=webpage.

The issuer was also CN=webpage, confirming it was self-signed.

This proved that HTTPS was working correctly even though the certificate was not trusted by the browser.

Errors Faced and Resolutions
Error 1: Site loaded on HTTP but not on HTTPS
Symptom

http://webpage loaded.

https://webpage did not load.

Cause

Port 443 was not published from the Nginx container to the host, so the browser could not reach HTTPS.

Resolution

Added "443:443" under the Nginx service in docker-compose.yml.

Restarted the containers.

Error 2: Confusion about whether the problem was a path issue
Symptom

The site was not initially available over HTTPS, so it seemed like the document root or file path might be wrong.

Cause

The actual issue was not the document root. The main issue was HTTPS exposure and SSL wiring, not the app path.

Resolution

Confirmed that root /var/www/html; in default.conf was correct.

Focused on SSL path mounting and port publishing instead.

Error 3: Browser showed NET::ERR_CERT_AUTHORITY_INVALID
Symptom

Browser warning page: “Your connection isn’t private”.

Error code: NET::ERR_CERT_AUTHORITY_INVALID.

Cause

The certificate was self-signed and not issued by a trusted public Certificate Authority, so the browser did not trust it by default.

Resolution

Confirmed with curl -vk https://webpage/ that TLS was actually working.

Used the browser’s advanced option to continue to the site for local testing.

Understood that this warning is expected in a lab/dev environment when using self-signed certificates.

Validation Performed
Validate running containers
bash
docker ps
Expected result:

Nginx container running.

Port mappings include both 80->80 and 443->443.

Validate certificate files exist inside the container
bash
docker exec -it nginx_web ls -l /etc/nginx/ssl
Expected result:

mysite.crt

mysite.key

Validate Nginx configuration
bash
docker exec -it nginx_web nginx -t
Expected result:

syntax is ok

test is successful.

Validate HTTPS response
bash
curl -vk https://webpage/
Expected result:

TLS handshake succeeds.

The certificate subject and issuer are shown.

HTTP 200 response is returned over HTTPS.

What Was Learned
A self-signed certificate enables HTTPS encryption, but browsers do not trust it automatically because it is not signed by a public CA.

A Virtual Host in Apache or a server {} block in Nginx is the configuration that uses the certificate, but it is not the certificate itself.
​

For HTTPS to work in Docker, it is not enough to create the certificate. The container must also:

listen on port 443,

have certificate files mounted correctly,

and publish port 443 to the host.

curl -vk is a reliable way to prove that TLS is functioning even when the browser warns about trust issues.

Final Outcome
The local PHP site was successfully configured to run over HTTPS using a self-signed TLS certificate in an Nginx Docker container. The browser displayed a trust warning as expected, and curl -vk https://webpage/ confirmed that the certificate was presented correctly and the application responded with HTTP 200 over TLS.
