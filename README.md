To set up a new vm:

```sh
apt update
apt install nginx git kakoune golang certbot python3-certbot-nginx

# Install Go
wget https://go.dev/dl/go1.21.7.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.7.linux-amd64.tar.gz
echo 'export PATH=/usr/local/go/bin:/root/go/bin:$PATH' >> /etc/profile
. /etc/profile

# Add nak, platform templates
go install github.com/fiatjaf/nak@latest
git clone https://github.com/coracle-social/platform.git
```

To set up a new service:

```sh
SUBDOMAIN=YOUR_SUBDOMAIN_HERE
PORT=YOUR_PORT_HERE

# Nginx
cat /root/platform/templates/nginx.conf \
  | sed s/SUBDOMAIN/$SUBDOMAIN/g \
  | sed s/PORT/$PORT/g \
  | > /etc/nginx/sites-available/$SUBDOMAIN.coracle.tools
ln -s /etc/nginx/sites-{available,enabled}/$SUBDOMAIN.coracle.tools
service nginx restart

# Systemd
cat /root/platform/templates/systemd.service | sed s/SUBDOMAIN/$SUBDOMAIN/g > /etc/systemd/system/$SUBDOMAIN-relay.service
service $SUBDOMAIN-relay start

# Add user
adduser $SUBDOMAIN

# Log in as user
su $SUBDOMAIN && cd ~

# Install bun, clone repos
curl -fsSL https://bun.sh/install | bash
git clone https://github.com/coracle-social/coracle.git
git clone https://github.com/coracle-social/triflector.git

# Set up coracle
cd ~/coracle
bun install
NODE_OPTIONS=--max_old_space_size=16384 bun run build

# Set up triflector
cd ~/triflector
go get
echo "PORT=$PORT" >> .env
```
