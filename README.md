# LocalNetwork — Personal setup with Tailscale

This repository contains a set of Bash scripts to manage access and backups between devices on a personal network (server, desktop/laptop, phone). The idea is to use Tailscale as a VPN to connect devices (hence the `100.x.x.x` addresses) and then use SSH/sshfs/scp to mount remote filesystems, copy files, and open SSH sessions.

This README explains the general workflow, prerequisites, and in particular how to use `config.sh`, which holds the variables required by the scripts.

## Repository contents

- `config_example.sh` — example configuration template.
- `config.sh` — actual configuration file (do not commit this if it contains sensitive data).
- `nssh` — wrapper to quickly open SSH sessions to server/desktop/phone (options `-s`, `-c`, `-t`).
- `ncp` — wrapper to copy files via scp between local and remote (also supports remote->local).
- `montaGgioDrive` — script to mount/unmount the server filesystem using `sshfs`.
- `montatelefono` — script to mount/unmount the phone storage using `sshfs`.

## Architecture and workflow

1. Install and authenticate Tailscale on all devices. Tailscale assigns addresses in the `100.x.x.x` range (used in `config.sh`).
2. The scripts source variables from `config.sh` (IPs, users, ports, local/remote paths).
3. Use `nssh` for quick SSH sessions, `ncp` to move files, and `sshfs` (used by `montaGgioDrive`/`montatelefono`) to mount remote directories locally.

## Prerequisites

- Tailscale installed and authenticated on all devices.
- SSH access (ssh server running on the remote devices).
- On the local machine: `ssh`, `scp`, `sshfs` (fuse), `fusermount` (to unmount), `ping`, `notify-send` (optional for desktop notifications).
- Set up SSH keys (recommended) or be ready to enter passwords when prompted.

On Debian/Ubuntu you can install required packages with:

```bash
sudo apt update
sudo apt install -y openssh-client sshfs iputils-ping libfuse2 libnotify-bin
```

On Arch Linux and derivatives (Manjaro, EndeavourOS, etc.):

```bash
sudo pacman -S openssh fuse2 sshfs inetutils libnotify
```

### Setting up mount directories

The scripts use directories under `/mnt/` to mount remote filesystems. You need to create these directories and ensure your user has proper permissions:

```bash
# Create mount points
sudo mkdir -p /mnt/server /mnt/telefono

# Change ownership to your user (replace USERNAME with your actual username)
sudo chown USERNAME:USERNAME /mnt/server /mnt/telefono

# Or alternatively, if you want to allow all users in your group to access the mounts
sudo chown root:users /mnt/server /mnt/telefono
sudo chmod 775 /mnt/server /mnt/telefono
```

## The `config.sh` file

`config.sh` contains all variables used by the scripts. Use `config_example.sh` as a template: copy it to `config.sh` and fill in your values.

List of variables (description):

- `PORT_TEL` — SSH port for the phone (example: `8022`).
- `PORT_STD` — standard SSH port for server/desktop (`22`).
- `IP_SERVER` — server address (Tailscale or LAN).
- `USER_SERVER` — server remote username.
- `IP_COMP` — desktop/laptop address.
- `USER_COMP` — desktop/laptop username.
- `IP_TEL` — phone address (Termux / ssh server on phone).
- `USER_TEL` — phone username (e.g. Termux user).
- `HOME_TEL` — remote home on the phone (used to build paths).
- `REMOTE_DIR_TEL` — remote directory on the phone to mount (e.g. Termux shared storage).
- `HOME_SERVER`, `HOME_COMP` — remote home directories on server/desktop.
- `LOCAL_DIR_SERVER` — local mountpoint for the server (e.g. `/mnt/server`).
- `LOCAL_DIR_TEL` — local mountpoint for the phone (e.g. `/mnt/telefono`).

Notes:
- Paths in `config.sh` are concatenated in the scripts (e.g. `$HOME_SERVER$PATHFILE`).
- Make sure local mount directories (`LOCAL_DIR_*`) exist and are writable by your user.

## Using the scripts (examples)

- nssh — open a quick SSH session

```bash
./nssh -s    # connect to the server
./nssh -c    # connect to the desktop/laptop
./nssh -t    # connect to the phone (uses PORT_TEL)
```

- ncp — copy files with scp

Basic usage (arguments follow the order: source (flag), file, destination (flag), where):

```bash
# Copy from local to server:
./ncp -c local/path -s remote/path

# Copy from server to local:
./ncp -s remote_file -c local_destination

# Note: the script currently does not support direct remote->remote copying (remote host to another remote host)
```

- montaGgioDrive / montatelefono — mount and unmount with sshfs

```bash
./montaGgioDrive -m   # mount the server at $LOCAL_DIR_SERVER
./montaGgioDrive -s   # unmount $LOCAL_DIR_SERVER

./montatelefono -m    # mount the phone at $LOCAL_DIR_TEL
./montatelefono -s    # unmount $LOCAL_DIR_TEL
```

The scripts use `sshfs` with options like `reconnect`, `ServerAliveInterval`, and preserve local uid/gid where applicable.

## Security and practical tips

- Do not commit `config.sh` to public repositories — add `config.sh` to `.gitignore`.
- Use SSH keys and disable password authentication where possible. To copy your public key to a remote:

- Secure `config.sh` with restrictive permissions:

```bash
chmod 600 config.sh
```

- If you use Tailscale, use the Tailscale-assigned addresses (100.x.x.x) like in `config.sh`. Check assigned IPs with `tailscale status` on each device.

## Troubleshooting (common issues)

- `ssh: connect to host ... port 22: Connection refused` — verify the SSH server is listening on the specified port and firewall rules allow it.
- `fusermount: mount failed` — ensure `sshfs` and `fuse` are installed and the local mount directory exists and is writable.
- `Permission denied (publickey)` — the public key is not present in the remote `~/.ssh/authorized_keys` or permissions are incorrect.
- `scp` fails with paths that contain spaces: wrap paths in quotes.

## Suggestions for future improvements

- Add logging to the scripts to help diagnose mount and scp issues.
- Support remote->remote copying by executing `scp` on an intermediate host (with confirmation and safety checks).
- Add a small incremental backup script (rsync over ssh) for the server.

---

Author: Alessandro Maggio
