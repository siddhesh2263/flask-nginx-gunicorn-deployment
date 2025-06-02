# Flask Deployment on a Linux Server with Nginx & Gunicorn

This guide will walk you through deploying your Flask application from your local machine to a production-ready Linux server. The deployment will be secured with SSH key-based authentication, a firewall (UFW), and run using Nginx and Gunicorn for better performance and reliability.

<br>

## Part 1 - Local setup vs server context:

Your current Flask site works locally, but to make it accessible on the internet, you’ll deploy it on your own Linux server. Although this setup requires more effort, it provides the most control and flexibility for performance, security, and scalability.

<br>

## Part 2 - Accessing the server and updating packages:

You'll access your Linux server using SSH (via PowerShell on Windows, for example) and the server’s IP address.
Once connected, it’s critical to update the package lists and upgrade existing packages to ensure your server has the latest security patches:

```
sudo apt update && sudo apt upgrade -y
```

<br>

## Part 3 - Changing the server's hostname:

For easier management, set a descriptive hostname on your server:

```
sudo hostnamectl set-hostname flask-server
```

Also, update the hostname in your /etc/hosts file:

```
sudo nano /etc/hosts
```

<br>

## Part 4 - Adding a limited user:

Instead of using the root account for daily work (which is risky), create a new limited user (I called it `sidrk`):

```
sudo adduser sidrk
```

Grant this user sudo privileges:

```
sudo adduser sidrk sudo
```

Log out and log back in as this new user to continue the setup securely.

<br>

## Part 5 - Setting up SSH key-based authentication:

To enhance security and avoid password-based logins (which are vulnerable to brute-force attacks), you’ll set up SSH key-based authentication.

On the server, create a `.ssh` directory in the user’s home directory:

```
mkdir ~/.ssh
```

On your local machine (for me Windows), generate an SSH key pair:

```
ssh-keygen -b 4096
```

You can leave the passphrase empty or set one for extra security.

Copy the public key to the server:

```
scp C:\Users\lenov\.ssh\id_ed25519.pub sidrk@10.0.0.44:~/.ssh/authorized_keys
```

On the server, set strict permissions for the `.ssh` directory and its contents:

```
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
```

Test the connection. You should be able to SSH into the server without entering a password:

```
ssh sidrk@10.0.0.44
```

<br>