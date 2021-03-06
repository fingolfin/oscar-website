# Technical info about the web server setup

This document describes how the OSCAR website hosting is set up, to help
people who need to troubleshoot it or migrate it to a new host.

## Requirements

- Ubuntu VM
- Apache 2
- PHP (only for the webhook)
- Jekyll

## Where it is

The server can be reached via SSH:

    ssh oscar.computeralgebra.de

There is a dedicated user `oscar` who owns a clone of the oscar-website
repository at

    /home/oscar/oscar-website

This clone is owned by `oscar` and the group `www-data` so that the web
server can access it as well. If anything goes wrong with these permissions,
they can be fixed via

  chown -R oscar:www-data /home/oscar/oscar-website

## Automatic updates via webhook

Whenever a change is pushed to the master branch of the oscar-website
repository, GitHub activates a webhook we provide via `webhook.php` at
<https://oscar.computeralgebra.de/webhook.php>.

The crucial bit is at the end of this .php file, where an empty file
`/tmp/oscar-website.trigger` is created. This is detected by a systemd unit
/etc/systemd/system/oscar-website.path (a copy of this file is in the
/etc/ directory of the oscar-website repo).

This then triggers /etc/systemd/system/oscar-website.service
(a copy of this file is in the etc/ directory of the oscar-website repo).

This finally executes etc/update.sh, which runs jekyll.


For authentication, we set a secret token in `/etc/apache2/sites-enabled/oscar.conf`
which looks like this:

    SetEnv GITHUB_WEBHOOK_SECRET "MY_SECRET"

with the actual secret key taking the place of `MY_SECRET`. The same value
must be entered in the GitHub settings at
<https://github.com/oscar-system/oscar-website/settings/hooks>.


## Troubleshooting

The following assumes you are logged in as root (resp. used `sudo` to become root)
on the webserver.

If updates stop working, a good first place to look at is this output of this:

    systemctl status oscar-website.service

This prints a log with extra info. However, it might also say "service not
found". In that case, make sure that `oscar-website.service` and
`oscar-website.path` are installed and enabled:

    cp /home/oscar/oscar-website/etc/oscar-website.* /etc/systemd/system/
    systemctl enable oscar-website.service oscar-website.path

A problem that sometimes happens (e.g. if one directly pokes into the git
clone) are broken file permissions which can impede further operations, such
as git pulling updates or jekyll updating the website. To fix these, run the
following as root:

    chown -R oscar:www-data /home/oscar/oscar-website
    chown -R oscar:www-data /var/www/oscar-website
