# Fastapi systemd integration

This how to will help you to run your Fastapi app as a systemd service 

## FastAPI Implementation

1. Setting Up the FastAPI Server

   Create a new Python project and install FastAPI and Uvicorn:

   ```
   sudo apt install python3-venv
   ```

   ```
   mkdir fastapi-intx-app-deploy
   ```

   ```
   cd fastapi-intx-app-deploy
   ```

   ```
   python3 -m venv venv
   ```

   ```
   source venv/bin/activate
   ```

   ```
   pip3 install fastapi uvicorn
   ```

2. Create a file named `main.py` with the following code:

   This is an example code. You should use it in your source code.

   ```
   from fastapi import FastAPI, HTTPException, Form
   import os
   import subprocess
   app = FastAPI()
   NGINX_SITES_AVAILABLE = "/etc/nginx/sites-available"
   NGINX_SITES_ENABLED = "/etc/nginx/sites-enabled"
   WEB_ROOT = "/var/www"
   @app.post("/api/deploy")
   def deploy(full_domain: str = Form(...), html_content: str = Form(...)):
   # Step 1: Create Nginx configuration file
   config_content = f"""
   server {{
       listen 80;
       server_name {full_domain};
       root {WEB_ROOT}/{full_domain};
       index index.html index.htm;
       location / {{
           try_files $uri $uri/ =404;
       }}
    access_log /var/log/nginx/{full_domain}.access.log;
       error_log /var/log/nginx/{full_domain}.error.log;
   }}
   """
    config_path = os.path.join(NGINX_SITES_AVAILABLE, full_domain)
    try:
        with open(config_path, "w") as f:
            f.write(config_content)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to write Nginx config: {e}")
    # Step 2: Create web directory and index.html
    web_dir = os.path.join(WEB_ROOT, full_domain)
    try:
        os.makedirs(web_dir, exist_ok=True)
        with open(os.path.join(web_dir, "index.html"), "w") as f:
            f.write(html_content)
        subprocess.run(["sudo", "chown", "-R", "ubuntu:ubuntu", web_dir], check=True)
        subprocess.run(["sudo", "chmod", "-R", "755", web_dir], check=True)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to create web directory: {e}")
    # Step 3: Create symbolic link
    enabled_link = os.path.join(NGINX_SITES_ENABLED, full_domain)
    try:
        if not os.path.exists(enabled_link):
            subprocess.run(["sudo", "ln", "-s", config_path, enabled_link], check=True)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to create symbolic link: {e}")
    # Step 4: Reload Nginx
    try:
        subprocess.run(["sudo", "systemctl", "reload", "nginx"], check=True)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to reload Nginx: {e}")
    # Step 5: Install SSL certificate
    try:
        subprocess.run(["sudo", "certbot", "--nginx", "-d", full_domain, "--non-interactive", "--reinstall"], check=True)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to install SSL certificate: {e}")
    return {"status": "success", "message": f"Domain {full_domain} deployed successfully."}
   ```
   
  
