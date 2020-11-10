# Jellyfin SELinux

## Description

This is an SELinux policy module for running Jellyfin within relatively strict confinement compared to the default `unconfined_service_t` domain. The author will be targeting RHEL 8 as the deployment enviornment.

### Affiliation

This project is not formally affiliated with the Jellyfin project in any way.

# Usage

## Content

Under the default policy, it is expected that your media will be labeled `jellyfin_content_t`, but the default `fcontext` does not map any directories to this domain.
In order to add media under this policy, you will have to execute the following command sequence:

```
# semanage fcontext -a -t jellyfin_content_t "/path/to/your/media(.*)?"
# restorecon -Rv /path/to/your/media
```

Alternatively, for a more permissive system you can allow all paths to be used as media folders by default. See **Booleans** below.

## Shared Memory

Jellyfin uses shared memory files located in `/dev/shm` that are by default labeled as `tmpfs_t`. These files may outlive the server, and the server does not have the necessary
permissions on the `tmpfs_t` type, so these files must be deleted or relabeled as `jellyfin_tmpfs_t` when transitioning a previously-unconfined server into SELinux confinement,
e.g., using one of the following commands:

```
# find /dev/shm -user jellyfin -exec chcon -t jellyfin_tmpfs_t {} \;
```
```
# find /dev/shm -user jellyfin -delete
```

## Ports

Jellyfin's default port usage consists of the following:

```
tcp      8096, 8920
udp      48265, 43628, 35146, 49996, 57354, 7359, 1900
```

Ports 8096 and 8920 are used for HTTP and HTTPS, respectively, although only the former will be listening in the as-shipped configuration for Jellyfin as of this writing.

The UDP ports have more or less internal uses and the application behaves nicely without allowing them in your firewall.

Each of these ports must be labeled as `jellyfin_port_t` under your SELinux policy, with the exception of port 1900/udp, which is already labeled as `ssdp_port_t`.
The policy has been written to allow Jellyfin to listen on ports labeled as `ssdp_port_t` for smoother automated deployment without making potentially-breaking changes
to the default policy, although best practice would be to remove the `ssdp_port_t` label and apply the `jellyfin_port_t` label in its place.

To add them manually, run the following commands:

```
# semanage port -a -t jellyfin_port_t -p tcp 8096
# semanage port -a -t jellyfin_port_t -p tcp 8920
# semanage port -a -t jellyfin_port_t -p udp 48265
# semanage port -a -t jellyfin_port_t -p udp 43628
# semanage port -a -t jellyfin_port_t -p udp 35146
# semanage port -a -t jellyfin_port_t -p udp 49996
# semanage port -a -t jellyfin_port_t -p udp 7359
# semanage port -d -p udp 1900
# semanage port -a -t jellyfin_port_t -p udp 1900
```

## Booleans

### `jellyfin_access_all_ro` (default: False)

This boolean allows Jellyfin read-only access to the entire filesystem. This does not override
Discretionary Access Control, so when the application is behaving normally it will only be able
to read paths that are either world-readable, owned by jellyfin and readable by their owner, or
group-owned by jellyfin and readable by their group.

One should set this boolean to true if one wishes to use media directories that one cannot or
will not label `jellyfin_content_t`.

### `allow_jellyfin_list_all_dirs` (default: True)

This boolean allows Jellyfin to list the contents of any directory on the system (again, not
overriding DAC; see above). This permission is used by the directory browser, and can safely
be disabled after one has set up all of one's library paths for a Jellyfin deployment.


### `allow_jellyfin_discover_samba` (default: True)

This boolean allows Jellyfin to search and read files within the `/etc/samba` directory.
It is suspected (though the author cannot presently verify), that Jellyfin uses this to
discover network drives.


### `allow_jellyfin_network_connect` (default: True)

This boolean allows Jellyfin to make arbitrary network connections on TCP ports. This is used
by Jellyfin to make network requests, e.g., for the purpose of fetching media metadata.
