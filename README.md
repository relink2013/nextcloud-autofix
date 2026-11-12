# nextcloud-autofix

A small background service for the [LinuxServer.io Nextcloud image](https://docs.linuxserver.io/images/docker-nextcloud/) that fixes the things that usually go wrong after Nextcloud or app updates. It's built for anyone who rarely opens the Nextcloud admin page and would otherwise find out weeks later.

It runs inside the Nextcloud container as an s6 service (LSIO's `custom-services.d`), so it starts and stops with the container. No cron job or extra container is needed.

## What it does

**After every update** (on each container start, since that's when the LSIO image upgrades Nextcloud, and whenever Nextcloud or an app changes version), it runs the usual follow-up commands:

- `occ upgrade`, but only if the database still needs it
- `db:add-missing-indices`, `db:add-missing-columns`, `db:add-missing-primary-keys`. These clear the "Database missing indices" warning.
- `maintenance:repair`, with `--include-expensive` after a version change
- `maintenance:mimetype:update-db` and `maintenance:mimetype:update-js`
- `app:update --all`, which you can turn off
- afterwards, it writes any setup warnings still shown on the admin page to its log

**Maintenance mode stuck on.** Every 10 minutes it checks maintenance mode. If it's been on for 30 minutes and nothing that needs it is running (an upgrade, a repair, the updater), it finishes any pending upgrade, turns maintenance mode off and notifies you. If it can't, it leaves maintenance mode on and tells you why.

**Apps turned off by an upgrade.** When Nextcloud upgrades to a new major version, it turns off every app whose installed version doesn't declare support for it, without telling you. This service notices which apps an upgrade turned off and turns them back on once a compatible version is installed. It uses the same version rule as Nextcloud and never forces it. Until then it looks for a newer version every 6 hours and keeps waiting. Apps you turn off yourself are left alone.

**Notifications.** Anything that needs a person is sent as a Nextcloud notification to the first admin, so it also shows up in the desktop and mobile apps.

Everything it does is logged to `/config/log/nextcloud-autofix.log` inside the container, and to the container log.

## Install

Your Nextcloud container needs a folder mounted at `/custom-services.d`. If it doesn't have one yet, add a volume such as:

```
/path/to/appdata/nextcloud/custom-services:/custom-services.d
```

Then, on the Docker host as root, replace the path with that folder and run:

```bash
D=/path/to/appdata/nextcloud/custom-services; mkdir -p "$D" && curl -fsSL https://raw.githubusercontent.com/relink2013/nextcloud-autofix/main/nextcloud-autofix -o "$D/nextcloud-autofix" && chown root:root "$D/nextcloud-autofix" && chmod 755 "$D/nextcloud-autofix"
```

Restart the container. The LSIO image only runs files in that folder if they're owned by root, which is why the command sets the owner.

To update, run the same command again and restart.

## Settings

All optional, set as environment variables on the Nextcloud container:

| Variable | Default | Meaning |
| --- | --- | --- |
| `AUTOFIX_CHECK_INTERVAL` | `600` | Seconds between checks |
| `AUTOFIX_STUCK_AFTER` | `1800` | Seconds maintenance mode may stay on before it counts as stuck |
| `AUTOFIX_NOTIFY_USER` | first admin | Nextcloud user who gets the notifications |
| `AUTOFIX_APP_UPDATES` | `1` | `0` turns off automatic app updates after Nextcloud updates |
| `AUTOFIX_APP_RETRY` | `21600` | Seconds between looks for a compatible version of an app an upgrade turned off |
| `AUTOFIX_NEVER_REENABLE` | (none) | Space-separated app ids never to turn back on |

## Keeping maintenance mode on deliberately

Create `/config/autofix.hold` inside the container (in your appdata folder, that's `.../nextcloud/autofix.hold` next to `www` and `log`). While that file exists, the service leaves maintenance mode alone. Delete it when you're done.

## Notes

- It only knows which apps an upgrade turned off from upgrades that happen after it's installed. Apps that were already off stay off until you turn them on.
- Its state is kept in `/config/autofix/`. Delete that folder to start fresh.
- It's written for the LinuxServer.io image: it relies on s6 (`with-contenv`, `s6-setuidgid`), the `abc` user and Nextcloud at `/app/www/public`.

## License

MIT
