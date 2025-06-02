# Flask Deployment on a Linux Server with Nginx & Gunicorn

This guide will walk you through deploying your Flask application from your local machine to a production-ready Linux server. The deployment will be secured with SSH key-based authentication, a firewall (UFW), and run using Nginx and Gunicorn for better performance and reliability.

<br>

## Local setup vs server context:

Your current Flask site works locally, but to make it accessible on the internet, you’ll deploy it on your own Linux server. Although this setup requires more effort, it provides the most control and flexibility for performance, security, and scalability.

<br>

## Accessing the server and updating packages:

You'll access your Linux server using SSH (via PowerShell on Windows, for example) and the server’s IP address.
Once connected, it’s critical to update the package lists and upgrade existing packages to ensure your server has the latest security patches:

```
sudo apt update && sudo apt upgrade -y
```

<br>

## Changing the server's hostname:

For easier management, set a descriptive hostname on your server:

```
sudo hostnamectl set-hostname flask-server
```

