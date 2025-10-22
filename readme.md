# Flask Deployment on a Linux Server with Nginx & Gunicorn

This guide will walk you through deploying your Flask application from your local machine to a production-ready Linux server. The deployment will be secured with SSH key-based authentication, a firewall (UFW), and run using Nginx and Gunicorn for better performance and reliability.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/all-merged.png?raw=true)

<br>

## 📝 Table of Contents

* [Part 0 - Assumptions](#part-0---assumptions)  
* [Part 1 - Local setup vs server context](#part-1---local-setup-vs-server-context)  
* [Part 2 - Accessing the server and updating packages](#part-2---accessing-the-server-and-updating-packages)  
* [Part 3 - Changing the servers-hostname](#part-3---changing-the-servers-hostname)  
* [Part 4 - Adding a limited user](#part-4---adding-a-limited-user)  
* [Part 5 - Setting up SSH key-based authentication](#part-5---setting-up-ssh-key-based-authentication)  
* [Part 6 - Disabling password authentication in SSH](#part-6---disabling-password-authentication-in-ssh)  
* [Part 7 - Firewall setup with UFW](#part-7---firewall-setup-with-ufw)  
* [Part 8 - Transferring the Flask project](#part-8---transferring-the-flask-project)  
* [Part 9 - Setting up Python virtual environment](#part-9---setting-up-python-virtual-environment)  
* [Part 10 - Managing environment variables securely](#part-10---managing-environment-variables-securely)  
* [Part 11 - Running Flask for testing](#part-11---running-flask-for-testing)  
* [Part 12 - Installing and configuring Nginx and Gunicorn](#part-12---installing-and-configuring-nginx-and-gunicorn)  
* [Part 13 - Adjust firewall rules for HTTP](#part-13---adjust-firewall-rules-for-http)  
* [Part 14 - Running Gunicorn](#part-14---running-gunicorn)  
* [Part 15 - Using Supervisor to keep Gunicorn running](#part-15---using-supervisor-to-keep-gunicorn-running)  
* [Part 16 - Final touches and Nginx configuration](#part-16---final-touches-and-nginx-configuration)  
* [Part 17 - How Nginx and Gunicorn work together](#part-17---how-nginx-and-gunicorn-work-together)
* [Part 18 - Future upgrades](#part-18---future-upgrades)
* [Part 19 - Guide checklist](#part-19---guide-checklist)
* [Part 20 - References](#part-20---references)

<br>

## Part 0 - Assumptions:

Before you begin, this guide assumes the following:

* You know how to install Ubuntu (or your chosen Linux distribution) and are comfortable using it.

* You already have a working Flask project that you want to deploy in a production environment.

<br>

## Part 1 - Introduction, Local setup vs server context:

Your current Flask site works locally, but to make it accessible on the internet, you’ll deploy it on your own Linux server. Although this setup requires more effort, it provides the most control and flexibility for performance, security, and scalability.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/server-flow.png?raw=true)

<br>

This diagram shows how a Flask app is deployed in production using Nginx and Gunicorn. Nginx handles HTTP requests and static files, forwarding dynamic requests to Gunicorn. Gunicorn runs multiple Flask workers and uses the WSGI protocol to execute Python code, ensuring scalability, security, and efficient request handling.

Below is an overview of some of the tools and setup we'll be using:

### Nginx:
Nginx is a high-performance web server that also acts as a reverse proxy. It efficiently handles static file delivery, such as CSS, JavaScript, and images, and forwards dynamic requests to Gunicorn for processing. Additionally, Nginx balances the load between multiple application workers to improve performance.

### Gunicorn:
Gunicorn is a Python WSGI HTTP server that runs multiple worker processes, each capable of handling requests to your Flask application. This design allows for parallel processing, better CPU utilization, and improved reliability for serving web requests.

### Supervisor:
Supervisor is a process control system that manages and monitors processes like Gunicorn. If a Gunicorn worker crashes or stops unexpectedly, Supervisor automatically restarts it, ensuring high availability and reliability of your Flask application.

### UFW (Uncomplicated Firewall):
UFW is a firewall configuration tool that simplifies the task of securing your server. It allows you to easily control incoming network connections, restricting them to only essential services like SSH and HTTP.

### Python Virtual Environment (venv):
Python’s built-in virtual environment tool, venv, creates an isolated environment for your Flask application. This ensures that project dependencies are kept separate from system-wide Python packages, providing better stability and easier management.

### SSH Key-based Authentication:
SSH key-based authentication is a secure alternative to password-based login. By using SSH keys, you make it much more difficult for attackers to gain access through brute-force password guessing.

### Config File (`/etc/config.json`):
A configuration file stored at `/etc/config.json` provides a secure location to store sensitive environment variables, such as secret keys and database URIs. This approach keeps sensitive information separate from your codebase for better security and maintainability.

<br>

## Part 2 - Accessing the server and updating packages:

You'll access your Linux server using SSH (via PowerShell on Windows, for example) and the server’s IP address.
Once connected, it’s critical to update the package lists and upgrade existing packages to ensure your server has the latest security patches:

```
sudo apt update && sudo apt upgrade -y
```

<br>

## Part 3 - Changing the server's hostname:

We’ll update your server’s hostname for easy identification. You’ll use the `hostnamectl` command and update the `/etc/hosts` file to reflect the new hostname consistently across the system:

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

You’ll harden your SSH configuration by disabling root login and password authentication. This ensures only your SSH keys are accepted, protecting against password-guessing attacks.

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

## Part 7 - Firewall setup with UFW (Uncomplicated Firewall):

We’ll configure UFW to restrict access to only the necessary ports, like SSH and HTTP. This firewall setup helps protect your server from unauthorized incoming traffic and ensures only known services are reachable.

Install and configure UFW to protect your server:

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

This section shows how to create a Python virtual environment to isolate your Flask project’s dependencies from system-wide packages. We’ll also activate the environment and install project dependencies from your `requirements.txt` file.

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

You’ll temporarily run your Flask app using `flask run` to make sure everything is working. This allows you to quickly verify that the transferred code and virtual environment are set up correctly:

```
export FLASK_APP=run.py
flask run --host=0.0.0.0
```

This allows external access. You can test by visiting the server’s IP and port in your browser.

<br>

## Part 12 - Installing and configuring Nginx and Gunicorn:

You’ll install Nginx as a reverse proxy and Gunicorn as the WSGI server. This part covers setting up Nginx to forward requests to Gunicorn and handle static files separately for better performance. Gunicorn will run the Python code.

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

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/nginx-config-forwarding.png?raw=true)

<br>

## Part 13 - Adjust firewall rules for HTTP:

You’ll open port 80 for HTTP traffic and remove any unnecessary testing ports from the firewall. This ensures your Flask app is accessible to the world while keeping your server secure.

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

You’ll learn how to start Gunicorn with multiple workers to handle concurrent requests. We’ll also discuss adjusting the number of workers based on your server’s available CPU cores.

Start Gunicorn with the recommended number of worker processes:

```
gunicorn -w 3 run:app
```

To check how many cores your server has:

```
nproc --all
```

Adjust the number of workers accordingly.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/gunicorn-pid.png?raw=true)

<br>

## Part 15 - Using Supervisor to keep Gunicorn running:

We’ll install Supervisor to manage Gunicorn as a background service. Supervisor will monitor Gunicorn, restarting it if it crashes and ensuring it stays up without manual intervention:

```
sudo apt install supervisor
```

Create a Supervisor configuration file:

```
sudo nano /etc/supervisor/conf.d/flaskblog.conf
```

Provide the full path to the Gunicorn executable inside the virtual environment.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/supervisor-config.png?raw=true)

<br>

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

We’ll cover final tweaks like adjusting Nginx’s `client_max_body_size` to allow larger file uploads. You’ll also restart Nginx to apply all the changes and confirm everything is set up for production.

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

This part explains how Nginx and Gunicorn work together in production. Nginx handles static files and load balancing, while Gunicorn runs multiple Flask app instances to serve dynamic requests in parallel.

![alt text](https://github.com/siddhesh2263/flask-nginx-gunicorn-deployment/blob/main/assets/forking-merged.png?raw=true)

<br>

Gunicorn creates multiple workers, each acting as a separate process running the Flask application. These processes are independent of each other and can handle incoming requests in parallel, effectively creating multiple instances of the app running simultaneously. Nginx acts as a reverse proxy sitting in front of Gunicorn, forwarding requests to one of the workers. By default, Nginx uses round-robin load balancing to distribute requests evenly across all available workers. This setup not only improves concurrency, allowing multiple requests to be served simultaneously, but also enhances reliability - if one worker crashes, other workers continue to handle traffic seamlessly. Additionally, Nginx manages static files like CSS, JavaScript, and images, further reducing the load on the Flask application code.

When a user is in a session and a worker crashes, the impact depends on how session data is stored. Flask’s default session mechanism stores session data in the user’s browser cookie, meaning no session data is lost as long as the server can still decode the cookie using the same `SECRET_KEY`. If a user refreshes the page or makes another request, the session cookie is sent to a different worker, preserving their logged-in state. However, any unsaved in-memory data, like a draft in an editor that hasn’t been committed to the database, would be lost if the worker crashes because that data exists only in the memory of that particular process. When the user submits the form and saves their work, the data is sent to the database and remains safe. Meanwhile, Gunicorn and process managers like Supervisor (or systemd) will automatically restart any crashed workers. Subsequent requests from the user will be handled by another running or restarted worker, ensuring that the overall user experience remains uninterrupted as long as the application doesn’t rely on in-memory state.

<br>

## Part 18 - Future upgrades:

To further enhance the production readiness of your Flask deployment, consider implementing the following upgrades:

* Use a Custom Domain Name: Point your domain to your server’s IP for a more professional and memorable application URL.

* Enable HTTPS with SSL/TLS: Secure communication between clients and your server by obtaining a free SSL certificate using Let’s Encrypt. This improves security and builds trust with users accessing your application.

<br>

## Part 19 - Guide checklist:

This final section summarizes the entire deployment process in a concise checklist. It ensures you have all the major components in place to deploy your Flask app securely and reliably in production:

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

<br>

## Part 20 - References:

* [All You Need to Know about WSGI](https://shorturl.at/Y36eZ)

* [Python Flask Tutorial: Deploying Your Application (Option #1) - Deploy to a Linux Server](https://shorturl.at/uFuj8)