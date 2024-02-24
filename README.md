To set up a new vm:

```sh
apt update
apt install nginx git kakoune golang certbot python3-certbot-nginx sqlite3

# Install Go
wget https://go.dev/dl/go1.21.7.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.7.linux-amd64.tar.gz
echo 'export PATH=/usr/local/go/bin:/root/go/bin:$PATH' >> /etc/profile
. /etc/profile

# Add nak, platform templates
go install github.com/fiatjaf/nak@latest
git clone https://github.com/coracle-social/platform.git

# Remove default nginx
rm /etc/nginx/sites-enabled/default
```

To set up a new service:

```sh
SUBDOMAIN=YOUR_SUBDOMAIN_HERE
PASSWORD=YOUR_PASSWORD_HERE
PORT=YOUR_PORT_HERE

# First, set up DNS for both relay.$SUBDOMAIN and $SUBDOMAIN, otherwise certbot will fail

# Set up swap
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Add user
adduser $SUBDOMAIN

# Log in as user
su $SUBDOMAIN
cd ~

# Install nvm, yarn, clone repos
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
. ~/.bashrc

# Clone repos
git clone https://github.com/coracle-social/coracle.git
git clone https://github.com/coracle-social/triflector.git

# Set up coracle
cd ~/coracle
nvm install
nvm use
npm install --global yarn
yarn
NODE_OPTIONS=--max_old_space_size=16384 yarn build

# Set up triflector
cd ~/triflector
go get
echo "PORT=$PORT" >> .env

# Nginx
cat /root/platform/templates/nginx.conf \
  | sed s/SUBDOMAIN/$SUBDOMAIN/g \
  | sed s/PORT/$PORT/g \
  > /etc/nginx/sites-available/$SUBDOMAIN.coracle.tools

# Certbot
certbot --nginx -d $SUBDOMAIN.coracle.tools -d relay.$SUBDOMAIN.coracle.tools

# Enable the site and restart nginx
ln -s /etc/nginx/sites-{available,enabled}/$SUBDOMAIN.coracle.tools
service nginx restart

# Back to root
exit

# Systemd
cat /root/platform/templates/systemd.service | sed s/SUBDOMAIN/$SUBDOMAIN/g > /etc/systemd/system/$SUBDOMAIN-relay.service
service $SUBDOMAIN-relay start
```
