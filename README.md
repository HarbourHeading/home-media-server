# Home Media Server

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Home Media Server](#home-media-server)
    * [Performance/Hardware](#performancehardware)
    * [Roadmap](#roadmap)
    * [Installation](#installation)
        + [Alpine Linux](#alpine-linux)
        + [Tailscale](#tailscale)
            - [ACL](#acl)
        + [Docker](#docker)
    * [Post Install](#post-install)
        + [Configure base URL](#configure-base-url)
            - [Jellyfin](#jellyfin)
            - [Prowlarr, Sonarr, Radarr & Lidarr](#prowlarr-sonarr-radarr--lidarr)
            - [qBittorrent](#qbittorrent)
        + [Tailscale serve](#tailscale-serve)
        + [Setup integrations](#setup-integrations)
            - [Jellyfin](#jellyfin-1)
            - [qBittorrent](#qbittorrent-1)
            - [Prowlarr](#prowlarr)
            - [Sonarr, Radarr & Lidarr](#sonarr-radarr--lidarr)
    * [Debugging](#debugging)
        + [Unable to add root folder - Folder '/tv/' is not writable by user 'abc'](#unable-to-add-root-folder---folder-tv-is-not-writable-by-user-abc)
    * [Tips](#tips)
        + [Expose setup to the internet using tailscale funnel](#expose-setup-to-the-internet-using-tailscale-funnel)

<!-- TOC end -->

A docker-compose setup of a media server. Covers viewing, installation, searching and managing various media. 
Installation shows a complete example setup on alpine linux, but other OS' should work fine as the core applications are run from docker-compose.

Setup includes by default:
- [Jellyfin](https://jellyfin.org/docs/): Media manager/server, alternative to Plex. Allows viewing media files like videos, images, music etc.
- [Prowlarr](https://wiki.servarr.com/prowlarr): Index manager/proxy for torrents. Lets other applications (e.g. sonarr and Radarr) search/query for media files from various torrent sites at once.
- [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr): Proxy server to bypass cloudflare protections. To be used with [Prowlarr for specific sources](https://wiki.servarr.com/en/prowlarr/faq#can-i-use-flaresolverr-indexers).
- [qBittorrent](https://www.qbittorrent.org/): Torrent manager/installer. To be used to actually download the torrents found by sonarr or radarr, queried to Prowlarr.
- [Sonarr](https://wiki.servarr.com/sonarr): TV/Series manager. Search and install movies through qbittorrent (install) and prowlarr (query torrents) through integrations.
- [Radarr](https://wiki.servarr.com/radarr): Movie manager. Search and install movies through qbittorrent (install) and prowlarr (query torrents) through integrations.
- [Lidarr](https://wiki.servarr.com/lidarr/quick-start-guide): Music manager. Search and install music through qbittorrent (install) and prowlarr (query torrents) through integrations.
- [Gluetun](https://github.com/qdm12/gluetun?tab=readme-ov-file#setup): Run VPN in a container, to be used by other containers as a network interface. Just saves a lot of hassle.
- [Docker](https://wiki.alpinelinux.org/wiki/Docker): To manage everything in containers. Allows for quick and contained deployment.

Adding another service to the list is easy. Just create another container for it. If you search for e.g. "kapowarr docker container" you should find [steps to add it to the docker-compose](https://casvt.github.io/Kapowarr/installation/docker/#__tabbed_4_2).
For inspiration and ideas, see [Awesome arr](https://github.com/Ravencentric/awesome-arr?tab=readme-ov-file) which lists many arr applications.

## Performance/Hardware
I am able to run it comfortably on **2 core, 4GB RAM, 4GB swap alpine linux virtual machine**. Even then, I believe you can get away with **1 core, 2GB RAM and 4GB swap**.
The more disk space, the merrier. Running on an HDD is fine. If you want a "better" setup, I would mount `/` on an SSD/NVME, 
then update the media files to be on a mounted HDD drive ([LVM](https://wiki.alpinelinux.org/wiki/Setting_up_Logical_Volumes_with_LVM) recommended for expansion).

**NOTE**: I have not tried streaming 4K 60fps videos or equivalent on this setup. The requirements may change based on your uses. 
[Jellyfin documentation](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/#supported-acceleration-methods) holds more information on it, where you may even want a dedicated GPU.

## Roadmap
- [ ] Setup subtitles service for movies/tv/anime
- [ ] Refine container user:group permissions
- [ ] Setup script to run (folders, permissions, boilerplate config with completed integrations etc.)
- [ ] Create TOC for README
- [ ] Setup simple health checks and dependency logic for all services

## Installation
### Alpine Linux
[Alpine linux](https://docs.alpinelinux.org/user-handbook/0.1a/Working/post-install.html) is a lightweight linux distro.
[Install](https://alpinelinux.org/downloads/) and boot into Alpine Linux Virtual Machine. Where you choose to install it up to you, 
for example on [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview).

Following steps assumes you are running as root (as sudo is not installed by default on alpine linux):
- Run `setup-alpine` and follow the prompts.
- Update to [edge packages](https://wiki.alpinelinux.org/wiki/Include:Upgrading_to_Edge).
- Run commands to install sudo and docker, as well as autostart it on boot.
````
apk add sudo docker docker-compose
rc-update add docker default
/etc/init.d/docker start
````
Don't forget to [add your user to sudoers](https://ostechnix.com/add-delete-and-grant-sudo-privileges-to-users-in-alpine-linux/). Otherwise, they cannot use sudo.
If you want to run docker commands without sudo, run `sudo addgroup <my-user> docker`.

### Tailscale
[Tailscale](https://tailscale.com/kb/1017/install) is "your own private network", in simpler terms. It can be used to access your services
outside your own local - home - network. Not strictly necessary, but here it serves the purpose of both allowing access outside your home network, and serving the content with HTTPS.
If you choose not to use tailscale, you will need to set up TLS/HTTPS and network access on your own. **DO NOT SKIMP OUT ON SETTING UP HTTPS. HTTP IS NOT SAFE!**

I will not go through setting up an entire tailscale network. There are [plenty of videos/documentation for that](https://tailscale.com/kb/1017/install). I will just give a working ACL and `tailscale serve` config to apply.

#### ACL
A minimum [tailscale ACL (JSON Editor)](https://login.tailscale.com/admin/acls/file) like below should work fine. When testing, you can set a grant for all IPs with `*` just to debug during the setup phase.
````json
{
    "tagOwners": {
    "tag:media-server":             ["my-email@example.com"],
    "tag:media-server-all-access":  ["my-email@example.com"]
    },

	"tests": [
		{
			"src":    "tag:media-server-all-access",
			"accept": ["tag:media-server:80", "tag:media-server:443"],
			"deny":   ["tag:media-server:12832"]
		}
	],
 
	"grants": [
		{
			"src": ["tag:media-server-all-access"],
			"dst": ["tag:media-server"],
			"ip":  ["tcp:80", "tcp:443"]
		}
	]
}
````
Give the media servers the tag `media-server` and your client `media-server-all-access`. You can separate them further as fits your needs.

Tailscale serve config is set up later in [Post Install > Tailscale Serve](#tailscale-serve).

### Docker
Assumes you have installed docker and docker-compose.
Run the application in the same directory as the docker-compose.yml file is in with
````
docker-compose up -d
````
Commands below are useful for debugging the containers.
````bash
docker ps                         # to see service status
docker logs <container>           # To see service logs. E.g. "docker logs jellyfin"
docker stats                      # Performance/usage of running containers
docker exec -it <container> bash  # Start bash shell in container to run other commands, e.g. "ls -lah" to check file permissions
````

## Post Install
Now we assume you can see all the containers when using `docker ps`, and they are healthy! First off, congrats!
You can access the HTTP version of the websites at (IPs/domains will differ from yours)

| Service     | URL                                |
|-------------|------------------------------------|
| Jellyfin    | `http://100.122.39.113:8096`       |
| qBittorrent | `http://100.122.39.113:8080`       |
| Prowlarr    | `http://100.122.39.113:9696`       |
| Sonarr      | `http://100.122.39.113:8989`       |
| Radarr      | `http://100.122.39.113:7878`       |
| Lidarr      | `http://100.122.39.113:8686`       |
Make sure you can access the web UI of each through your browser. Verify the tailscale ACL or any firewalls aren't blocking you, and check docker logs if issues arise.
Now, we will update the base URL in the configuration of each service, to afterward setup HTTPS using [tailscale serve](https://tailscale.com/kb/1242/tailscale-serve).

**NOTE**: qBittorrent credentials for first sign-in can be found in logs. Use `docker logs qbittorrent`. Afterward, change it in `Tools > Options > WebUI > Authentication`.

### Configure base URL
**NOTE**: Upon making changes to base url for each service below, it will be inaccessible before you run the corresponding tailscale serve command.
If you make a mistake, most (if not all) services have config files you can edit on the host, then just restart the container to revert it.
You may want to create a backup before continuing, just in case.

#### Jellyfin
Go to http://100.122.39.113:8096, follow the setup wizard and when setting media paths, set them as `/media/movies`, `/media/tv`, `/media/music` etc.
It automatically detects changes in these files. These files are added by other services, e.g. radarr (movies), sonarr (tv) & lidarr (music).

Once in the main dashboard, click on your profile in the top-right, and click on `Administration > Dashboard`.
Here, Find on the left sidebar `Advanced > Networking > Server Address Settings > Base URL` and set it to `/jellyfin` and enable HTTPS.

#### Prowlarr, Sonarr, Radarr & Lidarr
All 4 have about the same UI. Create an account and follow the same steps for all of them, replacing the `URL Base value` to their respective service names prefixed with `/`, for example `/lidarr` for lidarr.

| Service     | URL                           |
|-------------|-------------------------------|
| Prowlarr    | `http://100.122.39.113:9696`  |
| Sonarr      | `http://100.122.39.113:8989`  |
| Radarr      | `http://100.122.39.113:7878`  |
| Lidarr      | `http://100.122.39.113:8686`  |

#### qBittorrent
Does not seem to support setting a base URL. Skip for now.

### Tailscale serve
[tailscale serve](https://tailscale.com/kb/1242/tailscale-serve) will set up the domain and url path with HTTPS for us. Saves a great deal of time.

Run commands

| Service     | Command                                                                                          | Notes                                                                             |
|-------------|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| Jellyfin    | `sudo tailscale serve --bg --https=443 --set-path /jellyfin http://127.0.0.1:8096/jellyfin`      | Requires Jellyfin "Base URL" = `/jellyfin`                                        |
| qBittorrent | `sudo tailscale serve --bg --https=443 http://127.0.0.1:8080`                                    | qBittorrent does **not** support Base URL / path prefix                           |
| Prowlarr    | `sudo tailscale serve --bg --https=443 --set-path /prowlarr http://127.0.0.1:9696/prowlarr`      | Set "Base URL" = `/prowlarr`                                                      |
| Sonarr      | `sudo tailscale serve --bg --https=443 --set-path /sonarr http://127.0.0.1:8989/sonarr`          | Set "Base URL" = `/sonarr`                                                        |
| Radarr      | `sudo tailscale serve --bg --https=443 --set-path /radarr http://127.0.0.1:7878/radarr`          | Set "Base URL" = `/radarr`                                                        |
| Lidarr      | `sudo tailscale serve --bg --https=443 --set-path /lidarr http://127.0.0.1:8686/lidarr`          | Set "Base URL" = `/lidarr`                                                        |

Now, you should be able to reach the pages using (updating the URL to your environment)

| Service     | URL                                                      |
|-------------|----------------------------------------------------------|
| Jellyfin    | `https://my-host.tail1c2ub3.ts.net/jellyfin`             |
| qBittorrent | `https://my-host.tail1c2ub3.ts.net`                      |
| Prowlarr    | `https://my-host.tail1c2ub3.ts.net/prowlarr`             |
| Sonarr      | `https://my-host.tail1c2ub3.ts.net/sonarr`               |
| Radarr      | `https://my-host.tail1c2ub3.ts.net/radarr`               |
| Lidarr      | `https://my-host.tail1c2ub3.ts.net/lidarr`               |

### Setup integrations
It is assumed you have all docker containers running and can be connected to with HTTPS for these steps. If using tailscale, you should've already had a look around to set the `base url` in configuration (either through files or in the web UI).

**NOTE**: Services accept very different references to other containers/integrations. Some accept `localhost`, some accept e.g. `100.122.39.113` (tailscale IP), some accept the url `https://my-host.tail1c2ub3.ts.net/prowlarr` etc.
Try different variations and run the test to figure out which one works.

Below are bare-minimum configuration to set up the integrations and have a working setup. I strongly recommend you look around in their respective documentation after you confirm it works.
**NOTE**: Service order is intentional.

#### Jellyfin
1. Go to https://my-host.tail1c2ub3.ts.net/jellyfin
2. follow the setup wizard and when setting media paths, set them as `/media/movies`, `/media/tv`, `/media/music` etc.

It automatically detects changes in these files. These files are added by other services, e.g. radarr (movies), sonarr (tv) & lidarr (music).

#### qBittorrent
1. Go to https://my-host.tail1c2ub3.ts.net. 
2. Change `Tools > Options > Advanced > qBittorrent Section > Network interface`, and set it to the VPN interface. Likely `tun0`.
3. Verify `Tools > Options > Downloads > Saving Management > Default Save Path` is set to `/downloads`, and incomplete torrents kept in `/downloads/incomplete`.
4. Tweak the rest to your personal preference. Default works fine.

#### Prowlarr
1. Go to https://my-host.tail1c2ub3.ts.net/prowlarr.
2. In `Settings > Download Clients` add qBittorrent and fill in the form.
3. In `Settings > Indexers > Indexer Proxies` add FlareSolverr and fill in the form.
4. In `Settings > Apps > Applications` add Lidarr, Radarr and Sonarr and fill in the form. 
   API keys are retrieved from the targeted applications web UI under `Settings > General > Security > API Key`.

#### Sonarr, Radarr & Lidarr
1. In `Settings > Media Management > Root Folders` add the corresponding root folder and fill in the form. If issues arise, check [debugging](#debugging), 
   e.g. [Folder '/tv/' is not writable by user 'abc'](#unable-to-add-root-folder---folder-tv-is-not-writable-by-user-abc).
2. In `Settings > Download Clients` add qBittorrent and fill in the form.
3. In `Settings > Media Management > Movie/Track/Episode Naming > Rename Movies/Tracks/Episodes` I recommend turning it on.

| Service     | URL                                                      | Root Folder |
|-------------|----------------------------------------------------------|-------------|
| Sonarr      | `https://my-host.tail1c2ub3.ts.net/sonarr`               | `/tv`       |
| Radarr      | `https://my-host.tail1c2ub3.ts.net/radarr`               | `/movies`   |
| Lidarr      | `https://my-host.tail1c2ub3.ts.net/lidarr`               | `/music`    |

## Debugging
### Unable to add root folder - Folder '/tv/' is not writable by user 'abc'
Permission errors related to [GitHub issue](https://github.com/linuxserver/docker-radarr/issues/30). Can appear on sonarr or radarr when adding `root folder`.
Happens because the mounted file is owned by `root:root` (check with `docker exec -it <container> bash -c "ls -lah <folder>"`) instead of `abc:users`.
To fix it, run associated
````bash
docker exec -it sonarr bash -c "chown -R abc:users /tv"
docker exec -it radarr bash -c "chown -R abc:users /movies"
docker exec -it lidarr bash -c "chown -R abc:users /music"
````
This is at best a workaround, however. A permanent solution would be a remake of the user:group permissions which is on the roadmap, or setup entrypoint which sets permissions before mounting.

## Tips
### Expose setup to the internet using tailscale funnel
If you want to expose your setup to the internet (not recommended, but ultimately up to you if you accept and understand the associated risks), an option to those who've already used tailscale to serve the website is [tailscale funnel](https://tailscale.com/kb/1223/funnel).
Not tried it myself, but should work wonders for this type of setup.
