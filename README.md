# Monitor Docker for Home Assistant

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg?style=for-the-badge)](https://hacs.xyz/docs/faq/custom_repositories/)

## About

Monitor Docker lets you track Docker hosts and their containers from Home Assistant, and start, stop and restart containers. It connects to the Docker daemon locally through its UNIX socket, or remotely over TCP with optional TLS.

This repository is a maintained fork of earlier versions of the integration; see [Credits and history](#credits-and-history).

## Preparing Docker

When Home Assistant runs in a Docker container, mount the Docker socket into it as `-v /var/run/docker.sock:/var/run/docker.sock`.

**docker run example**
```
docker run -d \
  ... \
  -v /var/run/docker.sock:/var/run/docker.sock \
  homeassistant/home-assistant
```

**docker-compose.yaml example**
```yaml
services:
  hass:
    image: homeassistant/home-assistant
    ...
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ...
```

NOTE: Mounting `/var/run/docker.sock` read-only (`:ro`) has no effect, because it is a socket and not a file.

Home Assistant OS and Supervised installations do not expose the Docker socket; see the [Q&A](#qa) for how to connect through a socket proxy.

### No memory values reported

If every memory sensor reads 0, the Docker API is not reporting memory usage, usually because the memory cgroup is disabled in the kernel.

**Raspberry Pi (Raspberry Pi OS)**
- Open `/boot/cmdline.txt` (on newer releases, `/boot/firmware/cmdline.txt`).
- Append `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to the end of the existing line. Do *not* add a new line or replace the existing one.
- Reboot.

**Ubuntu / Debian**
- Open `/etc/default/grub`.
- Change `GRUB_CMDLINE_LINUX_DEFAULT=""` to `GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"`.
- Run `sudo update-grub`.
- Reboot.

NOTE: The Ubuntu/Debian steps are untested; use them at your own risk.

## Installation

### HACS (recommended)

The `Monitor Docker` entry in the HACS default catalog installs the original ualex73 version, not this fork. To install this fork, add it as a custom repository:

1. In HACS, open the ⋮ menu and choose **Custom repositories**.
2. Add `https://github.com/eseverson/hass_monitor_docker` with the type **Integration**.
3. Search for **Monitor Docker** and download it.
4. If applicable, mount `/var/run/docker.sock` into your Home Assistant container (see [Preparing Docker](#preparing-docker)).
5. Restart Home Assistant.

### Manual

1. Copy the `custom_components/monitor_docker` directory into `<config dir>/custom_components/`.
2. If applicable, mount `/var/run/docker.sock` into your Home Assistant container.
3. Restart Home Assistant.

## Configuration

Monitor Docker is set up entirely from the UI. YAML configuration is no longer supported, and a `monitor_docker:` block in `configuration.yaml` is ignored.

1. Go to **Settings → Devices & services → Add integration** and select **Monitor Docker**.
2. Set up the connection to the Docker host (fields below).
3. Select which containers to monitor.
4. Select which conditions to monitor for the host and for each container.

Add the integration once per Docker host, each with its own name.

### Connection

| Field | Description |
| ----- | ----------- |
| Name | Name of this Docker host, used as the prefix of its entity IDs. Must be unique. Defaults to `Docker`. |
| URL | Leave empty to use the local socket (`unix:///var/run/docker.sock`). For a remote daemon, use `tcp://ip:2375`, `http://ip:2375` or `https://ip:2376`. Do not end the URL with a slash. SSH is not supported. |
| Update interval | Seconds between updates. Defaults to 10. |
| Certificate path | Directory inside Home Assistant holding `ca.pem`, `cert.pem` and `key.pem`. Setting it enables TLS for a TCP connection; an `http://` or `tcp://` URL is switched to `https://`. |
| Retry interval | Seconds to wait before reconnecting after a TCP error. Defaults to 60. |

### Containers

| Field | Description |
| ----- | ----------- |
| Containers | The containers to monitor. Leave empty to monitor all of them, including containers created later. |
| Ignore ephemeral containers | When monitoring all containers, skip those Docker marks as ephemeral: started with `--rm`, or with restart policy `no`. Useful for one-off `docker run` containers that would otherwise each leave a device behind. |

### Changing the configuration

Open the integration under **Settings → Devices & services**, then choose **Reconfigure** from the ⋮ menu to change the monitored containers or conditions. The connection settings cannot be changed afterwards; to change them, remove the integration and add it again.

Devices for containers that no longer exist, or are no longer monitored, are removed automatically when the integration loads. You can also delete a container's device by hand from its device page.

## Entities

Each Docker host becomes a device, and each monitored container becomes a device linked to its host. Every entity has a unique ID, so you can rename entities and devices in the UI.

Each container device has:
- a **switch** that starts (on) and stops (off) the container: `switch.<name>_<container>`
- a **Restart container** button: `button.<name>_<container>_restart`
- one **sensor** per selected container condition: `sensor.<name>_<container>_<condition>`

The host device has one sensor per selected host condition: `sensor.<name>_<condition>`.

For example, with the default name `Docker`, the CPU sensor of a container named `jellyfin` is `sensor.docker_jellyfin_cpu`.

### Host conditions

The defaults are marked with ✓.

| Condition | Description | Unit | Default |
| --------- | ----------- | ---- | :-----: |
| version | Docker version | - | ✓ |
| containers_running | Number of running containers | - | ✓ |
| containers_paused | Number of paused containers | - | |
| containers_stopped | Number of stopped containers | - | |
| containers_total | Total number of containers | - | |
| containers_cpu_percentage | CPU usage of all containers. This scales with the number of cores: with 8 cores, it can reach 800% | % | ✓ |
| containers_1cpu_percentage | CPU usage of all containers, scaled to 0–100% | % | |
| containers_memory | Memory usage of all containers | MiB | ✓ |
| containers_memory_percentage | Memory usage of all containers | % | |
| images | Number of images | - | |

### Container conditions

| Condition | Description | Unit | Default |
| --------- | ----------- | ---- | :-----: |
| state | Container state: created, restarting, running, removing, paused, exited or dead | - | |
| status | Container status, e.g. `Up 13 days` or `Exited (0) 11 hours ago` | - | ✓ |
| health | Container health, if the container defines a health check | - | |
| uptime | Container start time | timestamp | ✓ |
| image | Container image | - | ✓ |
| image_hash | ID of the container's image | - | |
| restart_count | Times the restart policy has restarted the container. Manual restarts are not counted, and it resets to 0 when the container is recreated | - | |
| cpu_percentage | CPU usage. This scales with the number of cores: with 8 cores, it can reach 800% | % | ✓ |
| 1cpu_percentage | CPU usage, scaled to 0–100% | % | |
| memory | Memory usage | MiB | ✓ |
| memory_percentage | Memory usage | % | |
| network_speed_up | Upload speed. **Not** available when the network mode is `host` or `none` | KiB/s | |
| network_speed_down | Download speed. **Not** available when the network mode is `host` or `none` | KiB/s | |
| network_total_up | Total uploaded. **Not** available when the network mode is `host` or `none` | MiB | |
| network_total_down | Total downloaded. **Not** available when the network mode is `host` or `none` | MiB | |
| allinone | Special condition: creates a single sensor per container that holds all the other selected conditions as attributes, **instead of** one sensor per condition | - | |

## Actions

`monitor_docker.restart` restarts a monitored container.

| Field | Description |
| ----- | ----------- |
| name | Name of the container (required). |
| server | Name of the Docker host, as entered during setup. Only needed when you have more than one host. |

```yaml
action: monitor_docker.restart
data:
  name: jellyfin
```

## Debugging

To enable debug logging, add the following to `configuration.yaml`, or use **Enable debug logging** on the integration's page:

```yaml
logger:
  logs:
    custom_components.monitor_docker: debug
```

## Q&A

1. **Question:** Does this integration work with Home Assistant OS or Supervised?  
   **Answer:** Yes, through a socket proxy. These installations do not expose the Docker UNIX or TCP socket, but you can run [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) on the host to expose it over TCP. Other users have tested and confirmed this setup, but I cannot provide support for it. Start the proxy with the following docker-compose file, then enter `http://<host_ip>:2375` as the URL when setting up the integration. `POST=1` is what allows the switches and buttons to start, stop and restart containers.
    ```yaml
    # Proxy the Docker socket so Monitor Docker can read container statistics
    services:
      dockerproxy:
        image: tecnativa/docker-socket-proxy
        container_name: dockerproxy
        privileged: true
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        ports:
          - 2375:2375
        environment:
          - BUILD=1
          - COMMIT=1
          - CONFIGS=1
          - CONTAINERS=1
          - DISTRIBUTION=1
          - EXEC=1
          - IMAGES=1
          - INFO=1
          - NETWORKS=1
          - NODES=1
          - PLUGINS=1
          - SERVICES=1
          - SESSION=1
          - SWARM=1
          - POST=1
    ```
2. **Error:** `Missing valid docker_host.Either DOCKER_HOST or local sockets are not available.`  
   **Answer:** The Docker socket is most likely not mounted in your Home Assistant container. Check that you added the `/var/run/docker.sock` volume.
3. **Error:** `aiodocker.exceptions.DockerError: DockerError(900, "Cannot connect to Docker Engine via http://10.0.0.1:2376...)`  
   **Answer:** You are connecting over TCP and the remote address cannot be reached. Test it with `docker -H tcp://10.0.0.1:2376 ps` (replace `10.0.0.1` with your IP address). Docker's [remote access guide](https://docs.docker.com/engine/daemon/remote-access/) has more help.
4. **Question:** Is the Docker TCP socket over TLS supported?  
   **Answer:** Yes. Use a URL such as `https://ip:2376` and set **Certificate path** to a directory inside Home Assistant that holds your `ca.pem`, `cert.pem` and `key.pem`. The certificate's hostname is not checked. For example, mount the certificates with docker-compose and enter `/certs` as the certificate path:
    ```yaml
    services:
      hass:
        image: homeassistant/home-assistant
        ...
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          # The files must be named "ca.pem", "cert.pem" and "key.pem"
          - ./certs:/certs
        ...
    ```
5. **Question:** Can this integration monitor 2 or more Docker hosts?  
   **Answer:** Yes. Add the integration once for each host, with a unique name and that host's URL. Each host has its own TLS setting, so TLS and plain TCP connections can be mixed.
6. **Question:** Can creating, deleting or recreating a container be added to the integration?  
   **Answer:** The Docker library this integration uses has no easy and safe way to do this. Use *docker compose* for these operations. If anybody can make this work safely, I'll be happy to merge the PR.
7. **Question:** Can you add more security to a switch?  
   **Answer:** Not from the integration. Restrict it on the dashboard instead, for example with a card's tap action confirmation, or a card such as [restriction-card](https://github.com/iantrich/restriction-card).
8. **Question:** All the reported memory values are 0 (zero). Can this be fixed in the integration?  
   **Answer:** No. The integration only reports what the Docker API provides, so the fix is in your Docker host; see [No memory values reported](#no-memory-values-reported).
9. **Question:** I get a permission denied error.  
   **Answer:** Docker and Home Assistant OS run as root and can always connect to `/var/run/docker.sock`. If you run Home Assistant in a Python venv or directly with Python, add the user that runs Home Assistant to the `docker` group, then reboot:
    ```
    $ sudo usermod -a -G docker <user>
    $ sudo reboot
    ```
10. **Question:** Can you add a check for image updates on e.g. Docker Hub?  
    **Answer:** That is outside the scope of Monitor Docker, and other tools already do it, such as [newreleases.io](https://newreleases.io) or [Diun](https://github.com/crazy-max/diun/).
11. **Question:** Is Docker over SSH supported?  
    **Answer:** No, the Docker library this integration uses does not support it. You may be able to get it working by forwarding the socket with `socat`; [this answer](https://serverfault.com/questions/127794/forward-local-port-or-socket-file-to-remote-socket-file/362833#362833) may help.
12. **Question:** Can I rename the entities?  
    **Answer:** Yes. Every entity has a unique ID, so you can rename it, or its device, in the UI. Entity IDs are built from the host name and the container name, so a container that is recreated with the same name keeps its entities and history.

## Credits and history

This is the fourth generation of this code base:

- **[Sander Huisman](https://github.com/Sanderhuisman/docker_monitor)** started it as Docker Monitor.
- **[Alexander (ualex73)](https://github.com/ualex73/monitor_docker)** rewrote it from threads to asyncio as Monitor Docker, and added much of the functionality documented here. His version is the one in the HACS default catalog.
- **[Getslow6](https://github.com/Getslow6/monitor_docker)** forked Alexander's repository and added a config flow (based on work by [dala318](https://github.com/ualex73/monitor_docker/issues/173)), updated the entity naming with devices per container and a switch as each device's main entity, and simplified the configuration. Setup through `configuration.yaml` was removed along the way.
- **[Evan Severson](https://github.com/eseverson/hass_monitor_docker)** maintains this fork, which adds:
  - the option to ignore ephemeral containers
  - automatic removal of stale container devices, and manual device deletion
  - registering devices for containers created after startup
  - valid, slugified entity IDs
  - linking container devices to their host
  - the `restart_count` condition
  - clean recovery when the Docker daemon restarts
  - reconfigure fixes that keep the saved containers and conditions
  - compatibility with Home Assistant 2026.11
  - brand images shipped with the integration

## License

Apache License 2.0
