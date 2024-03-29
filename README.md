# Nuxt Universal (SSR) Deployment Guide

A basic guide on how to deploy Nuxt.js Universal (SSR) app on a Vultr server.

----

## Step 1: Create new server on Vultr

1. Create new server Choose Cloud Compute

2. Server Location: Sydney

3. Server type: Debian 10 x64

4. Server Size: 25 GB SSD $5/mo

5. Add SSH key: In terminal `cat ~/.ssh/id_rsa.pub | pbcopy` and then select not to ask for password on login

6. Deploy

----

## Step 2: Connecting and setting up server via SSH

1. When it's done, you'll need to copy the server IP
- switch back to your terminal and run `ssh root@SERVER_IP`
- replace `SERVER_IP` with whatever the ip is for example `ssh root@45.63.31.54`

2. First thing, make sure the software is up to date
- Run `apt update` then
- Run `apt upgrade`

3. Add security
- To protect your server from randoms brute force logging in run `apt install fail2ban`

4. Install Nodejs
- Run `apt install nodejs`
- When that's done run `node -v` and check that it tells you a version, like `10.x`
> Note: If modules dependencies require newer versions (i.e. 16.x.x of `node` use `nvm` to download and manage `node`: 
> https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-debian-10

5. Install Yarn
-  Run `apt install yarnpkg` 
-  Running `yarn -v` will throw error, that's because on debian `yarn` is called `yarnpkg` so we'll fix that by linking it
-  Check where its install `which yarnpkg` should return: `/usr/bin/yarnpkg`
-  run `ln -s /usr/bin/yarnpkg /usr/local/bin/yarn`
that's saying to create a `yarn` which links to `yarnpkg`
- if that worked, you can now try `yarn -v` and confirm it's working: `1.13.0`
> Note: If using `nvm` install `yarn` with the below:
> https://classic.yarnpkg.com/en/docs/install#debian-stable

6. Install Git
- Run `apt install git`

----

## Step 3: Check your shh config

1. In a local terminal window `cd ~/.ssh/` and look for `config`
2. `nano config` and check `Host *` has `ForwardAgent yes` :
```
Host *
 AddKeysToAgent yes
 UseKeychain yes
 IdentityFile ~/.ssh/id_rsa
 ForwardAgent yes

Host 68.183.104.106
  ForwardAgent yes
```

This means that if you `ssh` in to your server, you can access your github repo from there. 
> Note: Make sure your connecting to GitHub via SSH by running `ssh -T git@github.com` locally. If its denied make sure to add your SSH Key to Github first.

----

## Step 4: Installing Apache on the server

1. Run `apt install apache2` 
> Note: Apache isn't being used just yet we'll set apache up to connect your domain name and then proxy to your nuxt server

----

## Step 5: Connect server to git and clone repo

1. Run `cd /var/www` which is kinda like your Sites directory
2. Run `git clone git@github.com:your_github_name/your_repo.git` for example `git clone git@github.com:studiolathe/ebb-dunedin.git`
3. Don't forget to manually create your `.env` on the server
> Note: If setting up staging branch, make sure to clone staging branch `git clone <url> --branch <branch> --single-branch [<folder>]` i.e `git clone git@github.com:studiolathe/flavedo-albedo.git --branch staging --single-branch flavedo-albedo-staging` 

----

## Step 6: Get your project running on the server

1. Cd into the project and then run `yarn install` which will install all the project dependancies
2. Then we just `yarn run build` and then `yarn start --hostname 0.0.0.0`
3. In order to have the Nuxt server running in the background we need to add `yarn global add pm2` and then run `pm2 start "yarn start --hostname 0.0.0.0"`
> Note: If setting up staging branch, make sure to run seperate PM2 instance `pm2 start "yarn start --hostname 0.0.0.0 --port 3001"` 

----

## Step 7: Create your deploy script in the local project folder

1. Create a file in the root of your project call `scripts/deploy.sh` and add the below, be sure to change `123.123.123.123` to the your server IP:

```
#!/usr/bin/env sh

USER=root
HOST=123.123.123.123
WEBROOT=/var/www/ebb-dunedin

ssh $USER@$HOST /bin/bash <<EOF
  cd $WEBROOT
  git fetch
  git reset --hard origin/master
  yarn install
  yarn run build
  pm2 restart all
EOF
```

2. Make the script executable (in your command line) `chmod +x ./scripts/deploy.sh`

----

## Step 8: Point your new IP to domain

1. Create a file in the root of your project `nano /etc/apache2/sites-available/project-folder.conf`

```
# Production
<VirtualHost *:80>
	ServerName your-domain-name.com
 	ServerAlias www.your-domain-name.com

	DocumentRoot /var/www/project-folder/dist

	ProxyPass / http://localhost:3000/
	ProxyPassReverse / http://localhost:3000/
</VirtualHost>

# Staging
<VirtualHost *:80>
	ServerName staging.flavedoandalbedo.com
 	ServerAlias www.staging.flavedoandalbedo.com

	DocumentRoot /var/www/flavedo-albedo-staging/dist

	ProxyPass / http://localhost:3001/
	ProxyPassReverse / http://localhost:3001/
</VirtualHost>
```

2. Then enable with `a2ensite project-folder`
3. Setup proxies `a2enmod proxy proxy_http`
4. Restart server `systemctl restart apache2`
5. In the domain providers DNS, create a new `A` record and point it at the new `IP` address and another `A` record with `www` as host that points to `@`
6 Check updated DNS via https://www.whatsmydns.net/
> Note: If reciving timeout errors it may be due to server firewalls. Run `apt purge ufw` and manage firewall via Vult UI. 

----

## Step 9: Add SSL certificate

1. Add certbot `apt install -y python3-certbot-apache`
2. Then run `certbot --apache`
3. Tell it to get a cert for both www and the normal domain
4. Say yes to enabling https redirect

----

## Step 10: Deploy your site

1. After pushing changes to git run `./scripts/deploy.sh` from the local project folder and watch it go!
