# sendspin_player

Installs [sendspin-cli](https://github.com/Sendspin/sendspin-cpp-cli), a headless
Sendspin player, so a Raspberry Pi becomes a target for Music Assistant or any
other Sendspin server. It is meant to run **next to MPD**: both play through the same
ALSA `dmix` PCM, so neither blocks the sound card.

The role downloads the release archive for the host's userland architecture and
verifies it against the release's `SHA256SUMS`. It then creates the unprivileged
`sendspin-cli` account (`systemd-sysusers`), renders `/etc/sendspin-cli.conf` and
enables the shipped systemd unit. The role also works in `--check` mode, including
on a host where sendspin-cli is not installed yet.

## Requirements

- **Debian / Raspberry Pi OS trixie.** The `arm64`, `armv7` and `x86_64` archives
  need glibc ≥ 2.38, and bookworm ships 2.36. The role checks this and stops with a
  clear message. Only the `armv6` archive also loads on bookworm.
- Facts must be gathered: the runtime package names (`libasound2t64` vs. `libasound2`)
  depend on `ansible_facts['distribution_release']`.
- **avahi-daemon must not be masked**, since servers find the player via mDNS. The
  collection's `rpi_avahi` role masks it by default: set `rpi_avahi_disable: false`
  for your Sendspin hosts. The role stops with a hint if avahi is masked; it doesn't
  unmask it behind another role's back.
- A `dmix` PCM in `/etc/asound.conf` that **two different users can share**. See below.
  The role does not manage `/etc/asound.conf`, since the sound card is host specific.

## dmix shared by MPD and sendspin-cli

MPD runs as `mpd`, and the player runs as `sendspin-cli`. dmix shares its buffer
through System V IPC, and the default permission is `0600`. The second user to open
the device then fails with "Permission denied" or "unable to create IPC semaphore".
Give the dmix a fixed key, don't salt it with the UID, and allow the `audio` group:

```
pcm.dmix {            # or whatever your existing dmix definition is called
    type dmix
    ipc_key 1024
    ipc_key_add_uid false
    ipc_gid audio
    ipc_perm 0660
    slave {
        pcm "hw:CARD=sndrpihifiberry,DEV=0"
        # rate 48000 ... keep your existing slave settings
    }
}
```

The shipped sysusers fragment already puts `sendspin-cli` into `audio`. Make sure
`mpd` is in `audio` too.

The player's sync accuracy depends on the delay ALSA reports. The extra
`plug` → `dmix` layer is usually fine. If one room stays audibly ahead or behind,
correct it with `sudo sendspin-cli delay <ms>` or from the server. Avoid
`sendspin_player_output_delay`, which only applies on the first run.

## Pairing (sendspin-cli ≥ 0.4.0)

Since 0.4.0 a server must pair with the player before it can play. A server that
hasn't paired can connect and shows the player, but playback does nothing. You
have three options:

- Paste the player's token into the server. `sudo sendspin-cli pair-token` prints it.
- Set a static `sendspin_player_pairing_code` (exactly 8 digits, **quoted** in YAML
  so a leading zero survives). Confirm each attempt with `sudo sendspin-cli pair confirm`.
- Set `sendspin_player_allow_unpaired: true` to let any server on the LAN play.

The pairings and the player's identity key live in the state directory
(`/var/lib/sendspin-cli`). See the [pairing wiki page](https://github.com/Sendspin/sendspin-cpp-cli/wiki/Pairing-a-Player).

## Role variables

| Name                                     | Default                    | Comment                                                                    |
| ---------------------------------------- | -------------------------- | -------------------------------------------------------------------------- |
| `sendspin_player_version`                | `0.4.0`                    | Release to install; bumping it reinstalls                                  |
| `sendspin_player_name`                   | `inventory_hostname_short` | Name shown in the controller                                               |
| `sendspin_player_output`                 | `plug:dmix`                | ALSA PCM; list options with `sendspin-cli -l`                              |
| `sendspin_player_port`                   | `8928`                     | Player's own WebSocket port (advertised via mDNS)                          |
| `sendspin_player_buffer_ms`              | `250`                      | Output queue, 10–2000 ms; raise it if you hear clicks                      |
| `sendspin_player_output_delay`           | `0`                        | First-run default only (see above)                                         |
| `sendspin_player_allow_unpaired`         | `false`                    | Let any server play without pairing (see above)                            |
| `sendspin_player_pairing_code`           | `""`                       | Static 8-digit pairing code; empty = fresh code per attempt                |
| `sendspin_player_fixed_volume`           | `false`                    | Full volume, server can't change it (line-out into an amp)                 |
| `sendspin_player_server`                 | `""`                       | Empty = wait for discovery; otherwise host, `ws://` URL or `mdns:[name]`   |
| `sendspin_player_id`                     | `""`                       | Empty = derived from the MAC address                                       |
| `sendspin_player_audio_format`           | `""`                       | Priority list, e.g. `flac:48000:24:2,pcm:48000:24:2`                       |
| `sendspin_player_input`                  | `""`                       | Capture device to stream to the server (mic / line-in)                     |
| `sendspin_player_hook_start` / `_stop`   | `""`                       | Shell commands run on stream start/stop                                    |
| `sendspin_player_log_level`              | `info`                     | `none` … `verbose`                                                         |
| `sendspin_player_extra_config`           | `{}`                       | Any other config key, e.g. `{"mdns-name": "Kitchen"}`                      |
| `sendspin_player_service_enabled`        | `true`                     | Enable and start the unit                                                  |
| `sendspin_player_state_on_tmpfs`         | `false`                    | State on `/run` to spare the SD card (see below)                           |
| `sendspin_player_disable_wifi_powersave` | `false`                    | NetworkManager drop-in; keeps mDNS reachable on Wi-Fi                      |
| `sendspin_player_check_glibc`            | `true`                     | Refuse to install on glibc < 2.38                                          |
| `sendspin_player_packages`               | see `defaults/main.yml`    | Runtime libraries; the names follow the distribution release               |

### State on tmpfs

`sendspin_player_state_on_tmpfs: true` moves the state directory to `/run/sendspin-cli`.
It survives service restarts but not a reboot. After a reboot the player has a
**new identity and no pairings**, and its volume, mute and delay are reset. Combine
it with `sendspin_player_allow_unpaired: true`, or be ready to re-pair after every
reboot.

### Wi-Fi power saving

Changing `sendspin_player_disable_wifi_powersave` restarts NetworkManager, which
briefly drops the network connection (and an SSH session over Wi-Fi).

### Pausing MPD when Sendspin starts

If you don't want MPD and Sendspin to play over each other, pause MPD from the start
hook. `mpc` must be installed, and MPD must accept connections on localhost:

```yaml
sendspin_player_hook_start: "mpc -q pause"
```

## Example playbook

```yaml
- name: MPD hosts that are also Sendspin players
  hosts: james_rpi
  collections:
    - oxivanisher.raspberry_pi
  roles:
    - role: oxivanisher.raspberry_pi.james_on_rpi
      tags: [james2]
    - role: oxivanisher.raspberry_pi.sendspin_player
      tags: [sendspin]
```

## Checking it

```
systemctl status sendspin-cli
journalctl -u sendspin-cli -f
sudo sendspin-cli status
sudo sendspin-cli pair-token                           # token to paste into the server
sudo -u sendspin-cli /usr/local/bin/sendspin-cli -l   # list outputs as the service user
```

## License

GPL-2.0-or-later

## Author Information

This role is part of the [oxivanisher.raspberry_pi](https://galaxy.ansible.com/ui/repo/published/oxivanisher/raspberry_pi/) collection, and the source for that is located on [github](https://github.com/oxivanisher/collection-raspberry_pi).
