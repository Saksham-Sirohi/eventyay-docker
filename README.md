# Setting Up a Working Environment for EventYay Components

This guide will help you set up the complete app.eventyay.com system using Docker.

## Prerequisites

- Docker (May require installation if not already done) / Install Docker Desktop on MacOS enable rosetta translation in docker if on arm / WSL integration to be used when using on Windows
- Docker Compose (Alternatively docker-compose can be used but requires editing command)
- Docker buildx
- NPM/Node (May require installation if not already done) on MacOS you can install it with homebrew
- git(ssh setup required)
- arch[i386(x86_64) as eventyay video npm install causes issues on arm based systems; use rosetta on mac]

**Check whether pre-requisites are installed on your system**

```bash
docker version
docker buildx version
docker compose version
npm -v
node -v
ssh -T git@github.com
arch
```

if any of these fail please check and install missing prerequisites before going further

## Video Tutorial

For visual guidance, you can watch the EventYay Setup tutorial for FOSSASIA Summit 2025:
[https://www.youtube.com/watch?v=w5pqJAnIG3M](https://www.youtube.com/watch?v=w5pqJAnIG3M)

## Initial Setup

### 1. Prepare Your Working Directory

First, set up a working directory where all development will happen:

```bash
# Define your working directory replace eventyay-dev with name of your preference
export WORKDIR=~/eventyay-dev
mkdir -p $WORKDIR
cd $WORKDIR
```

> **Note on Environment:** For optimal performance, use Ubuntu 22.04 LTS or newer(Prefer LTS as some latest builds are causing some issue with docker) , or macOS. Windows Subsystem for Linux (WSL) may cause issues with Docker networking and file permissions.

> **If you're on MacOS use terminal with rosetta to avoid arm specific errors if any, you can do this by checking open using rosetta this can be found in Applications/Utilities right-click terminal and select get info you will find the option there after enabling quit and restart terminal**

### 2. Clone Repositories

Clone all the required repositories:

```bash
# Define GitHub paths
FOSSASIA_GITHUB=https://github.com/fossasia

# Clone all FOSSASIA repos
git clone $FOSSASIA_GITHUB/eventyay-docker
git clone $FOSSASIA_GITHUB/eventyay-talk
git clone $FOSSASIA_GITHUB/eventyay-tickets
git clone $FOSSASIA_GITHUB/eventyay-video
```

> **Note:** Forking repositories is optional but recommended if you plan to contribute. If you've forked the repositories, you can add your personal clone as a remote **(Only for Development)** :
>
> ```bash
> # from the for loop you can remove repositories that you havent forked for development also remember to add your github username
> PERSONAL_GITHUB=git@github.com:<YOUR_GITHUB_USERNAME>
> for repo in eventyay-talk eventyay-tickets eventyay-video eventyay-docker; do
>   cd $WORKDIR/$repo
>   # this will remove old personal remote you can remove this if needed
>   git remote remove personal 2>/dev/null || true
>   git remote add personal $PERSONAL_GITHUB/$repo.git
>   git fetch personal
>   cd $WORKDIR
> done
> ```

All checked out repos (except eventyay-docker) should default to the `development` branch. For new development, check out a branch from the up-to-date `development` branch.

## Building and Configuration

### 1. Build Docker Images

```bash
# From your $WORKDIR
cd eventyay-talk && make local
cd ../eventyay-tickets && make local
cd ../eventyay-video && make local
cd ..
```

### 2. Set Up EventYay Video

The video webapp needs node modules etc installed and build. This is done in during the docker image build step, but since we mount the checked out eventyay-video directory into the container, the built-in directory with node-modules etc is hidden. If doing this on mac make sure rosetta is used by terminal you can confirm this by using arch this should be i386

```bash
cd eventyay-video/webapp
npm ci --legacy-peer-deps
NODE_OPTIONS=--openssl-legacy-provider npm run build
cd ../..
```

### 3. Create Data Directories

Ensure you are inside the `eventyay-docker` directory:

```bash
cd eventyay-docker
```

Create necessary data directories:
*Timestamp: [15:51](https://youtu.be/w5pqJAnIG3M?si=1QOQw-tIhuPXBjlR&t=951)*

```bash
mkdir -p data/{postgres,talk,ticket,video,video-webapp}
mkdir -p data/{talk,ticket,video}/data
mkdir -p data/video-webapp/public
chown -R $(id -u):$(id -g) data/
```

### 4. Configure Environment Variables

Create a `.env` file from the example if you don't need to make any changes to environment variables:

```bash
cp .env.example .env
```

### 5. Set Up Host Entries

**->Add the following entries to your `/etc/hosts` file:**

```
grep -qxF "127.0.0.1  app.eventyay.com" /etc/hosts || echo "127.0.0.1  app.eventyay.com" | sudo tee -a /etc/hosts
grep -qxF "127.0.0.1  video.eventyay.com" /etc/hosts || echo "127.0.0.1  video.eventyay.com" | sudo tee -a /etc/hosts
```

Alternatively, you can change all hostnames in the configuration files in `docker-compose.yaml` and `config/**`.

**->WSL users should run command prompt in order to add entries to Windows hosts:**

```
findstr /C:"127.0.0.1  app.eventyay.com" %SystemRoot%\System32\drivers\etc\hosts >nul 2>&1 && echo Already exists: app.eventyay.com || (echo 127.0.0.1  app.eventyay.com>>%SystemRoot%\System32\drivers\etc\hosts && echo Added: app.eventyay.com)

findstr /C:"127.0.0.1  video.eventyay.com" %SystemRoot%\System32\drivers\etc\hosts >nul 2>&1 && echo Already exists: video.eventyay.com || (echo 127.0.0.1  video.eventyay.com>>%SystemRoot%\System32\drivers\etc\hosts && echo Added: video.eventyay.com)
```

**->WSL users can alternatively install chrome inside wsl which will show as a seperate browser in windows but there may be performance issues**

```
cd /tmp
sudo wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt install --fix-broken -y
sudo dpkg -i google-chrome-stable_current_amd64.deb
```

Chrome in WSL can be accessed by (if installed inside WSL):

```
google-chrome app.eventyay.com
```

## Launching the Application (Ensure you are in eventyay-docker for this the whole time)

### 1. Start the Containers

```bash
docker compose -f docker-compose-dev.yml up -d
```

### 2. Initial User Setup

⚠️ **Critical:** Use identical email address and password for all the superusers. The systems are integrated and rely on matching credentials. Avoid root access as it can cause configuration issues.

### Create ticket superuser in the eventyay-ticket container

```bash
# Execute this command in your host terminal to enter the ticket container
docker exec -ti eventyay-ticket bash
```

> Once inside the container, run:
>
> ```bash
> # Create a superuser account for the ticket system
> cd ~
> pretix createsuperuser
> # Type 'exit' when finished to return to your host terminal
> exit
> ```

**IMPORTANT: Do not access the web pages yet!**

### Create talk superuser in the eventyay-talk container

```bash
# Execute this command in your host terminal to enter the talk container
docker exec -ti eventyay-talk bash
```

> Once inside the container, run:
>
> ```bash
> # Initialize talk system with a superuser account
> pretalx init
> # Type 'exit' when finished to return to your host terminal
> exit
> ```

### Create video superuser for the eventyay-video container

> ```bash
> # Initialize video system with a superuser account
> docker exec -it eventyay-video python3 manage.py migrate
> docker exec -it eventyay-video python3 manage.py import_config sample/worlds/sample.json
> docker exec -it eventyay-video python3 manage.py createsuperuser
> ```

   **Change the port number from 8443 to 8375 in config.js (Only for Local Development Setup)**

> ```bash
> # Initialize video system with correct config ensure you are in eventyay-docker directory
> CONFIG_FILE="../eventyay-video/webapp/config.js"
> OLD_PORT="8443"
> NEW_PORT="8375"
> if [[ ! -f "$CONFIG_FILE" ]]; then
>   echo "File not found: $CONFIG_FILE"
>   exit 1
> fi
> if grep -q "$OLD_PORT" "$CONFIG_FILE"; then
>   if [[ "$OSTYPE" == darwin* ]]; then
>     # macOS
>     sed -i '' "s/$OLD_PORT/$NEW_PORT/g" "$CONFIG_FILE"
>   else
>     # Linux
>     sed -i "s/$OLD_PORT/$NEW_PORT/g" "$CONFIG_FILE"
>   fi
>   echo "Port changed from $OLD_PORT to $NEW_PORT in $CONFIG_FILE"
> else
>   echo "Port $OLD_PORT not found in $CONFIG_FILE. No changes made."
> fi
> ```

## Accessing the System

Visit `https://app.eventyay.com/tickets/` and log in with the user/password you defined above.

## Setting Up SSO entry

Upon successful login turn on admin mode in the menu with your email go to admin under global settings in the side menu navigate to generate keys for sso use this redirect URL `https://app.eventyay.com/talk/oauth2/callback/`

```bash
# Execute this command in your host terminal to enter the talk container
docker exec -ti eventyay-talk bash
```

> Once inside the container, run:
>
> ```bash
> # Initialize talk system with a superuser account
> cd src
> ./manage.py create_social_apps
> # Type 'exit' when finished to return to your host terminal
> exit
> ```

After logging in to tickets, you should be able to go to `https://app.eventyay.com/talk/`, click on the login text, and get automatically logged in.

In order to access to access eventyay-video go to `http://app.eventyay.com:8002/control` **(Only for Local Development)** otherwise go to `http://app.eventyay.com/video`

## Development Workflow

- Changes to Python code and templates should be reflected immediately in the docker containers and visible in the browser after container reload, if not immediately.
- Changes to JavaScript pages may require rebuilds (Not Much Info Here).

## Troubleshooting

If you encounter issues with the login or connections between services, ensure:

1. All containers are running (`docker ps`).
2. The host entries in `/etc/hosts` are correct.
3. You've properly initialized both the ticket and talk systems with identical credentials.
4. If using Windows WSL, consider switching to a native Ubuntu 22.04+ installation for better Docker compatibility.
5. Docker networking can sometimes be problematic with WSL - if you experience connectivity issues, try restarting the Docker service.
6. If you are having problem with user creation due to data directory permissions it is highly likely that you have done something wrong in step 3 of building and configuration.
