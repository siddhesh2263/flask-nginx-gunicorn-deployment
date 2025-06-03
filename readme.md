# Flask Deployment on a Linux Server with Nginx & Gunicorn

This guide will walk you through deploying your Flask application from your local machine to a production-ready Linux server. The deployment will be secured with SSH key-based authentication, a firewall (UFW), and run using Nginx and Gunicorn for better performance and reliability.

<br>

## Part 0 - Assumptions:

Before you begin, this guide assumes the following:

* You know how to install Ubuntu (or your chosen Linux distribution) and are comfortable using it.

* You have basic terminal skills (e.g., navigating directories, editing files with nano or vim, running commands with sudo).

* You already have a working Flask project that you want to deploy in a production environment.

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

## Part 6 - Disabling password authentication in SSH

To prevent password-based logins completely and strengthen security, edit the SSH configuration file:

```
sudo nano /etc/ssh/sshd_config
```

Update these values:

```
PermitRootLogin no
PasswordAuthentication no
```

Restart the SSH service to apply the changes:

```
sudo systemctl restart sshd
```

<br>

## Part 7 - Firewall setup with UFW:

Install and configure UFW (Uncomplicated Firewall) to protect your server:

```
sudo apt install ufw
```

Set default rules:

```
sudo ufw default allow outgoing
sudo ufw default deny incoming
```

Explicitly allow SSH and Flask development port (5000):

```
sudo ufw allow ssh
sudo ufw allow 5000
```

Enable the firewall and confirm:

```
sudo ufw enable
sudo ufw status
```

<br>

## Part 8 - Transferring the Flask project:

You can either:

1. Clone the project from GitHub directly on the server.

2. Or copy the project directory from your local machine to the server using scp.

We'll go with option 2:

```
scp -r C:\Users\lenov\projects_2024\flask_corey_linux sidrk@10.0.0.44:~/
```

<br>

## Part 9 - Setting up Python virtual environment on the server:

We will need to run the Flask application inside a virtual environment. Install the below Python tools to set it up:

```
sudo apt install python3-pip
sudo apt install python3-venv
```

Create a virtual environment in your project directory:

```
python3 -m venv flask_corey_linux/venv
```

Activate it:

```
source flask_corey_linux/venv/bin/activate
```

Install your project’s dependencies (after ensuring you have a `requirements.txt`):

```
pip install -r requirements.txt
```

<br>

## Part 10 - Managing environment variables securely:

Instead of using environment variables directly (which can be tricky with different servers), create a JSON config file:

```
sudo touch /etc/config.json
sudo nano /etc/config.json
```

Add sensitive values like `SECRET_KEY`, `SQLALCHEMY_DATABASE_URI`, and email credentials in JSON format.

Update your Flask app to read from this config file using Python’s `json` module, replacing any `os.environ.get()` calls with `config.get()`.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/config-read-secret.png?raw=true)

<br>

## Part 11 - Running Flask for testing:

For initial testing, you can temporarily run Flask directly:

```
export FLASK_APP=run.py
flask run --host=0.0.0.0
```

This allows external access. You can test by visiting the server’s IP and port in your browser.

<br>

## Part 12 - Installing and configuring Nginx and Gunicorn:

For production, Nginx will act as a reverse proxy, and Gunicorn will run the Python code.

Install Nginx:

```
sudo apt install nginx
```

Install Gunicorn in your virtual environment:

```
pip install gunicorn
```

Remove the default Nginx configuration:

```
sudo rm /etc/nginx/sites-enabled/default
```

Create a new Nginx config file for your Flask app:

```
sudo nano /etc/nginx/sites-enabled/flaskblog
```

Configure it to forward Python requests to Gunicorn running on port 8000 and handle static files.

<br>

## Part 13 - Adjust firewall rules for HTTP:

Open port 80 for Nginx:

```
sudo ufw allow http/tcp
```

Remove development port 5000:

```
sudo ufw delete allow 5000
```

Apply the changes:

```
sudo ufw enable
```

Restart Nginx:

```
sudo systemctl restart nginx
```

<br>

## Part 14 - Running Gunicorn:

Start Gunicorn with the recommended number of worker processes:

```
gunicorn -w 3 run:app
```

To check how many cores your server has:

```
nproc --all
```

Adjust the number of workers accordingly.

<br>

## Part 15 - Using Supervisor to keep Gunicorn running:

Install Supervisor to ensure Gunicorn stays running and restarts on crashes:

```
sudo apt install supervisor
```

Create a Supervisor configuration file:

```
sudo nano /etc/supervisor/conf.d/flaskblog.conf
```

Provide the full path to the Gunicorn executable inside the virtual environment.

Create log directories:

```
sudo mkdir -p /var/log/flaskblog
sudo touch /var/log/flaskblog/flaskblog.err.log
sudo touch /var/log/flaskblog/flaskblog.out.log
```

Reload Supervisor to apply:

```
sudo supervisorctl reload
```

<br>

## Part 16 - Final touches and Nginx configuration:

If you need to increase file upload size in Nginx (to avoid errors like `413 Request Entity Too Large`), update:

```
sudo nano /etc/nginx/nginx.conf
```

Inside the `http` block, add:

```
client_max_body_size 5M;
```

Restart Nginx:

```
sudo systemctl restart nginx
```

The Flask app is now running in production, accessible by visiting your server’s IP address in a browser. Nginx is handling incoming traffic and static files, while Gunicorn runs your Flask code in the background, monitored by Supervisor for reliability.

<br>

## Part 17 - Closing points - How Nginx and Gunicorn work:

Gunicorn creates multiple workers, each acting as a separate process running the Flask application. These processes are independent of each other and can handle incoming requests in parallel, effectively creating multiple instances of the app running simultaneously. Nginx acts as a reverse proxy sitting in front of Gunicorn, forwarding requests to one of the workers. By default, Nginx uses round-robin load balancing to distribute requests evenly across all available workers. This setup not only improves concurrency, allowing multiple requests to be served simultaneously, but also enhances reliability—if one worker crashes, other workers continue to handle traffic seamlessly. Additionally, Nginx manages static files like CSS, JavaScript, and images, further reducing the load on the Flask application code.

When a user is in a session and a worker crashes, the impact depends on how session data is stored. Flask’s default session mechanism stores session data in the user’s browser cookie, meaning no session data is lost as long as the server can still decode the cookie using the same `SECRET_KEY`. If a user refreshes the page or makes another request, the session cookie is sent to a different worker, preserving their logged-in state. However, any unsaved in-memory data, like a draft in an editor that hasn’t been committed to the database, would be lost if the worker crashes because that data exists only in the memory of that particular process. When the user submits the form and saves their work, the data is sent to the database and remains safe. Meanwhile, Gunicorn and process managers like Supervisor (or systemd) will automatically restart any crashed workers. Subsequent requests from the user will be handled by another running or restarted worker, ensuring that the overall user experience remains uninterrupted as long as the application doesn’t rely on in-memory state.

<br>

## Part 18 - Guide checklist:

Below is the concise deployment roadmap for Flask on Linux: secure SSH, firewall, environment isolation, Nginx-Gunicorn setup, and Supervisor management.

```
# Deployment Checklist for Flask App on Linux Server

✅ Prepare your Linux server:
- Connect via SSH.
- Update and upgrade system packages.

✅ Change the server hostname:
- Use `hostnamectl` and update `/etc/hosts`.

✅ Create a limited user:
- Add a new user and give them `sudo` access.

✅ Configure SSH key-based authentication:
- Generate SSH key pair.
- Copy public key to the server.
- Set permissions for `.ssh`.
- Disable password authentication for SSH.

✅ Configure the firewall (UFW):
- Allow essential ports (SSH, HTTP).
- Deny all other incoming traffic.

✅ Transfer Flask project to the server:
- Use `scp` or clone from GitHub.

✅ Set up a Python virtual environment:
- Install `python3-venv` and `pip`.
- Create and activate the virtual environment.
- Install dependencies from `requirements.txt`.

✅ Configure environment variables:
- Create `/etc/config.json` with sensitive settings.
- Update Flask code to read from this file.

✅ Test run the Flask app:
- Use `flask run --host=0.0.0.0` for testing.

✅ Install and configure Nginx and Gunicorn:
- Install Nginx and remove default config.
- Create a new Nginx site config.
- Install Gunicorn in the virtual environment.
- Run Gunicorn with multiple workers.

✅ Adjust firewall rules for production:
- Allow HTTP traffic on port 80.
- Remove testing port 5000 from the allowed list.

✅ Use Supervisor to manage Gunicorn:
- Install Supervisor.
- Create Supervisor config for Gunicorn.
- Create log directories and files.
- Reload Supervisor to apply changes.

✅ Tweak Nginx for file upload limits:
- Update `client_max_body_size` in `nginx.conf`.

✅ Final verification:
- Visit the server’s IP address to confirm the app is live.
- Ensure the app stays running after SSH logout.

```