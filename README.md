To set up a new vm:

```sh
# Set up swap
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Install deps
apt update
apt install nginx git kakoune golang certbot python3-certbot-nginx sqlite3 ack rsync postgresql postgresql-contrib

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

To set up a new user:

```sh
USERNAME=username
PASSWORD=$(head -c18 /dev/urandom | base64)

adduser $USERNAME
echo $USERNAME:$PASSWORD | chpasswd
```

To set up a triflector relay:

```sh
PORT=5001
HOME_PATH=/home/$USERNAME
RELAY_PATH=$HOME_PATH/triflector
RELAY_DOMAIN=relay.example.com
RELAY_SERVICE="${RELAY_DOMAIN//./-}"

echo -n "First, set up DNS records, otherwise certbot will fail. Press Enter to continue."; read

# Create database
sudo -u postgres createdb $USERNAME;
sudo -u postgres psql -c "CREATE USER \"$USERNAME\" WITH PASSWORD '$PASSWORD';"

# Set up triflector
git clone https://github.com/coracle-social/triflector.git $RELAY_PATH

# Update env variables
echo "PORT=$PORT" >> $RELAY_PATH/.env
echo "DATABASE_URL=postgres://$USERNAME:$PASSWORD@localhost:5432/$USERNAME?sslmode=disable" >> $RELAY_PATH/.env

# Update permissions
chown -R $USERNAME $RELAY_PATH

# Install dependencies and build the app
sudo -u $USERNAME bash -c "cd ~/triflector && go get && go build ."

# Nginx
cat /root/platform/templates/relay.nginx.conf \
  | sed s/DOMAIN/$RELAY_DOMAIN/g \
  | sed s/PORT/$PORT/g \
  > /etc/nginx/sites-available/$RELAY_DOMAIN

# Certbot
certbot --nginx -d $RELAY_DOMAIN

# Enable the site and restart nginx
ln -s /etc/nginx/sites-{available,enabled}/$RELAY_DOMAIN
service nginx restart

# Create the systemd config file
cat /root/platform/templates/systemd.service \
  | sed s/USERNAME/$USERNAME/g \
  | sed s/DOMAIN/$RELAY_DOMAIN/g \
  | sed s/RELAY_PATH/$RELAY_PATH/g \
  | sed s/RELAY_SERVICE/$RELAY_SERVICE/g \
  > /etc/systemd/system/$RELAY_SERVICE.service

# Start the service
service $RELAY_SERVICE start
```

To add a flotilla instance:

```sh
PORT=5001
RELAY_DOMAIN=relay.example.com
FLOTILLA_DOMAIN=chat.example.com

# Log in as user
DOMAIN=$DOMAIN PORT=$PORT su $USERNAME

# Go to home dir
cd ~

# Install nvm
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
. ~/.bashrc

# Set up flotilla
git clone https://github.com/coracle-social/flotilla.git
cd ~/flotilla
nvm install
nvm use
echo "VITE_PLATFORM_RELAYS=wss://relay.$SUBDOMAIN.coracle.tools" >> .env.local
npm i
NODE_OPTIONS=--max_old_space_size=16384 npm run build
```

To remove a service:

```sh
SUBDOMAIN=mysubdomain

service $SUBDOMAIN-relay stop
rm /etc/systemd/system/$SUBDOMAIN-relay.service

rm /etc/nginx/sites-enabled/$SUBDOMAIN.coracle.tools
rm /etc/nginx/sites-available/$SUBDOMAIN.coracle.tools
service nginx restart

sudo -u postgres dropdb $SUBDOMAIN;
sudo -u postgres psql -c "DROP USER IF EXISTS \"$SUBDOMAIN\";"

rm -rf /home/$SUBDOMAIN
userdel $SUBDOMAIN
```
