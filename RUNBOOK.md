# Dojo Jump EC2 Hosting Runbook

This runbook shows three ways to host the dojo-jump app on an EC2 instance `Nginx` `Apache` `Flask`.

The app in this folder is a static site:

- `index.html`
- `style.css`
- `main.js`

That means you do not need Node, Flask, or a database on the EC2 instance. You only need a web server that can serve static files over port `80`.

## Assumptions

- The EC2 instance is already running.
- The EC2 security group allows inbound `22` and `80`.
- You have the `.pem` file for the EC2 key pair.
- You are running commands from your local directory with the dojo-jump files. 
- Replace `PUBLIC_IP` with the public IP of the target EC2 instance. 
- Replace `PATH_TO_KEY.pem` with your actual key path.

For the EC2 instance where the AMI is Amazon Linux 2023. Use `ec2-user` as the default SSH username.

Get the public IP if needed:

```bash
aws ec2 describe-instances \
  --instance-ids <target-instance-id-here> \
  --region us-east-1 \
  --query "Reservations[].Instances[].PublicIpAddress" \
  --output text
```

SSH into the instance:

```bash
ssh -i PATH_TO_KEY.pem ec2-user@PUBLIC_IP
```

Example:

```bash
ssh -i ~/.ssh/ec2-troy.pem ec2-user@98.80.139.147
```

Copy the app files to the EC2 instance:

```bash
scp -i PATH_TO_KEY.pem index.html style.css main.js ec2-user@PUBLIC_IP:/tmp/
```

Example:

```bash
scp -i ~/.ssh/ec2-troy.pem index.html style.css main.js ec2-user@98.80.139.147:/tmp/
```
## Which Option To Pick

- Use `Nginx` if you want the most common EC2 static-site setup.
- Use `Apache` if you want a traditional web server with familiar virtual host configuration.
- Use `Flask` if you want a small Python web app instead of a generic file server.

## Important Notes

- Only run one of these web servers on port `80` at a time.
- If you switch methods, stop and disable the previous web server first.

Examples:

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx

sudo systemctl stop httpd
sudo systemctl disable httpd

sudo pkill -f dojo-jump-app.py
```

- This app currently references some external assets over plain `http`. Serving the page over `http://PUBLIC_IP` will work. If you later enable HTTPS, those asset URLs should be updated to HTTPS or hosted locally to avoid mixed-content issues.
## Method 1: Nginx

This is the most common choice for serving static sites on EC2.

### Generic Nginx steps

Install Nginx:

```bash
sudo dnf update -y
sudo dnf install -y nginx
```

Create an app directory and copy the files into place:

```bash
sudo mkdir -p /usr/share/nginx/html/dojo-jump
sudo chown -R ec2-user:ec2-user /usr/share/nginx/html/dojo-jump
sudo cp /tmp/index.html /tmp/style.css /tmp/main.js /usr/share/nginx/html/dojo-jump/
```

Create an Nginx site config:

```bash
sudo tee /etc/nginx/conf.d/dojo-jump.conf >/dev/null <<'EOF'
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html/dojo-jump;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
EOF
```

Disable default config if present:

```bash
sudo rm -f /etc/nginx/default.d/*.conf
```

Test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
```

Verify:

```bash
curl http://localhost
```

## Method 2: Apache HTTP Server

Apache is a good alternative if you want a traditional web server with simple static hosting.

Install Apache:

```bash
sudo dnf update -y
sudo dnf install -y httpd
```

Create an app directory and copy the files into place:

```bash
sudo mkdir -p /var/www/dojo-jump
sudo cp /tmp/index.html /tmp/style.css /tmp/main.js /var/www/dojo-jump/
```

Create an Apache virtual host:

```bash
sudo tee /etc/httpd/conf.d/dojo-jump.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/dojo-jump

    <Directory /var/www/dojo-jump>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/dojo-jump-error.log
    CustomLog /var/log/httpd/dojo-jump-access.log combined
</VirtualHost>
EOF
```

Enable the site and disable the default site:

```bash
sudo rm -f /etc/httpd/conf.d/welcome.conf
```

Test and restart Apache:

```bash
sudo apachectl configtest
sudo systemctl enable httpd
sudo systemctl restart httpd
```

Verify:

```bash
curl http://localhost
```

## Method 3: Flask

Flask is a lightweight Python web framework. This version serves the same static dojo-jump files through a small Flask app.

Install Python tools and Flask:

```bash
sudo dnf update -y
sudo dnf install -y python3 python3-pip
python3 -m venv /home/ec2-user/dojo-jump-venv
/home/ec2-user/dojo-jump-venv/bin/pip install flask
```

Create an app directory and copy the files into place:

```bash
sudo mkdir -p /var/www/dojo-jump
sudo cp /tmp/index.html /tmp/style.css /tmp/main.js /var/www/dojo-jump/
```

Create the Flask app:

```bash
cat >/home/ec2-user/dojo-jump-app.py <<'EOF'
from flask import Flask, send_from_directory

app = Flask(__name__, static_folder="/var/www/dojo-jump")


@app.route("/")
def index():
    return send_from_directory("/var/www/dojo-jump", "index.html")


@app.route("/<path:path>")
def static_files(path):
    return send_from_directory("/var/www/dojo-jump", path)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
EOF
```

Start Flask on port `80`:

```bash
sudo bash -c 'nohup /home/ec2-user/dojo-jump-venv/bin/python /home/ec2-user/dojo-jump-app.py >/home/ec2-user/dojo-jump-flask.log 2>&1 &'
```

Verify:

```bash
curl http://localhost
```


## Cleanup

Use this section if you want to cleanly remove one hosting method before switching to another.

Stop all possible web services and free port `80`:

```bash
sudo systemctl stop nginx || true
sudo systemctl stop httpd || true
sudo pkill -f dojo-jump-app.py || true
```

Disable services from starting on reboot:

```bash
sudo systemctl disable nginx || true
sudo systemctl disable httpd || true
```

Remove method-specific config files if needed:

```bash
sudo rm -f /etc/nginx/conf.d/dojo-jump.conf
sudo rm -f /etc/httpd/conf.d/dojo-jump.conf
rm -f /home/ec2-user/dojo-jump-app.py
```

Optional cleanup of copied app files and logs:

```bash
sudo rm -rf /usr/share/nginx/html/dojo-jump
sudo rm -rf /var/www/dojo-jump
rm -f /home/ec2-user/dojo-jump-flask.log
```

## Troubleshooting

Check whether the web server is running:

```bash
sudo systemctl status nginx
sudo systemctl status httpd
ps -ef | grep '[d]ojo-jump-app.py'
```

Check whether port `80` is listening:

```bash
sudo ss -tulpn | grep :80
```

Check the EC2 firewall path:

- Security group must allow inbound TCP `80` from `0.0.0.0/0`
- The instance must be in the public subnet
- The public subnet route table must send `0.0.0.0/0` to the internet gateway

If the site works on `curl http://localhost` from inside the instance but not from your browser, the issue is almost always networking or the EC2 security group.