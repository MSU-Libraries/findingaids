# Finding Aids

Dockerized ArchivesSpace implementation that includes MSUL customizations
based off of the [Docker method](https://docs.archivesspace.org/administration/docker/)
of deployment provided by ArchivesSpace.

* [First Time Setup](#first-time-setup)
* [Upgrading](#upgrading)
* [Maintaining the Server](#maintaining-the-server)
* [Backup and Recovery](#backup-and-recovery)
* [Developer Notes](#developer-notes)
* [Non-MSU Users of this Repository](#non-msu-users-of-this-repository)

## First Time Setup

Install Docker on the target server and create a deploy user and group
that the CI user will be able to connect as.

Install required dependencies:

```bash
sudo apt install rsync
sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
sudo chmod a+x /usr/local/bin/yq
```

Create a directory for the SSL certificates at `/etc/ssl`. The expected files are:

```bash
findingaids.key
findingaids_lib_msu_edu_cert.cer
```

To request a new SSL certificate or update an existing one:

```bash
# Create the CSR
openssl req -new -newkey rsa:2048 -nodes -keyout findingaids.key -out findingaids.csr
# MSU users: Submit a ticket in TDX to get a new certificate using that CSR
cat findingaids.csr
# MSU users: Download the "Certificate only, PEM encoded" and "Root/Intermediate(s) only, PEM encoded" from the email
# Create the combined ssl certificate
cat findingaids_lib_msu_edu_interm.cer >> findingaids_lib_msu_edu_cert.cer
# Copy the ssl cert files to correct path
cp findingaids.key /etc/ssl/
cp findingaids_lib_msu_edu_cert.cer /etc/ssl/
```

Log rotation can be handled by Docker, if configured in the `/etc/docker/daemon.json`:

```bash
{
    "log-driver": "local",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    }
}
```

But we should also add rotation for the database backups. Create this file `/etc/logrotate.d/findingaids`:

```bash
/home/deploy/archivesspace/backups/logrotate_dummy.log {
  daily
  rotate 0
  create
  ifempty
  lastaction
      /bin/ls -t /home/deploy/archivesspace/backups/db_backup_*.tgz 2>/dev/null | /usr/bin/tail -n +4 | /usr/bin/xargs -r /bin/rm -f
  endscript
}

```

Running the CI pipeline (or the `deploy` script) will create the environment,
but if you are creating this from an existing environmnent you will need
to populate the database (which will trigger the Solr index to be rebuilt).

```bash
cat archivesspace.sql | docker exec -i mysql mysql -uas -pas123 archivesspace
# Restart the docker containers to have the missing migrations applied
sudo -Hu deploy /home/deploy/findingaids/deploy -v --skip-setup -a v4.1.1
```

### If using the GitLab CI/CD to deploy

Required CI/CD variables, remembering that different CI environments
can have unique values for these variables.

* `AS_VERSION`: The version of ArchivesSpace to use, e.g. `v4.1.1`
* `CONF_FILE`: The entire `config/config.rb` file to be used

Create a deploy user that will have read-only access to the repository to 
pull changes.  

```
adduser deploy
passwd -l deploy
adduser deploy docker
```

### If not using the GitLab CI/CD to deploy

You will need to create a `config/config.rb` file that will contain all your
local settings.

Build and deploy the site using the `deploy` script:

```bash
./deploy -t /path/to/deploy/files -a v4.1.1 -v
```

## Upgrading

* Update the `CONF_FILE` CI variable with the updated contents.
  This can be done by downloading the GitHub Docker asset from
  [the release page](https://github.com/archivesspace/archivesspace/releases)
  for example [archivesspace-docker-v4.1.1.zip](https://github.com/archivesspace/archivesspace/releases/download/v4.1.1/archivesspace-docker-v4.1.1.zip)
  doing a `vimdiff` on the `config/config.rb` and the customized config.
  Once all changes are made, copy the updated customized config back into
  the CI variable. `vimdiff custom-config.rb archivesspace/config/config.rb`.

* Update the `proxy-config/default.conf` with any changes we want to pull
  in from upstream comparing using the same method in the first step above.

* Update the `AS_VERSION` CI variable with the new version number.

* You may need to make updates to the files within `msul-theme`, specifically
  the `en.yml` files should be compared against the `en.yml` files in the 
  version being upgraded to in order to ensure that there are no new translation
  values being added that we need to merge into our local file. Be sure to
  compare `local/en.yml` and the `local/enums/en.yml` in both the `frontend`
  and `public` directories.


## Maintaining the Server

In the event the service goes down, you can try re-deploying the containers
either via the CI pipeline (re-run the last sucessfull job to the environment
or start a new pipeline from the default branch). Or to get things up quicker,
you can run the deploy script on the sever.

```bash
ssh findingaids.lib.msu.edu
sudo -Hu deploy /home/deploy/findingaids/deploy -a v4.1.1 --skip-setup -v
```

## Backup and Recovery

See the [official documentation](https://docs.archivesspace.org/administration/backup/)
for all the steps to take a manual backup and restore from it.

Backups will be automatically taken and stored in `./backups` directory.

## Developer Notes

### Testing the [API](https://archivesspace.github.io/archivesspace/api)

```bash
curl -X POST https://findingaids.lib.msu.edu/staff/api/users/[user]/login?password=[PASSWORD]

```

### Changing the source code

Update the `docker-compose.yml` file to add a command:

```bash
services:
  app:
    image: archivesspace/archivesspace:${ARCHIVESSPACE_DOCKER_TAG}
    command: sleep inf
```

Then redeploy the service:

```bash
docker compose up --detach
```

Connect to the container and get a copy of the source code, make your modifications, then run it:

```bash
docker exec -it archivesspace bash
mkdir /repo
cd /repo
wget https://github.com/archivesspace/archivesspace/archive/refs/tags/v4.1.1.zip
unzip v4.1.1.zip
archivesspace-4.1.1/build/run bootstrap
# make any modifications you want at this point and make sure the deploy script has the correct path
# such as putting in print statements such as: `Otherwiseputs("hi")` or `puts(my_var.to_s)` or `puts(my_obj.inspect)`
/build_and_deploy frontend
```

### java.lang.IllegalArgumentException: HOUR_OF_DAY: 2 -> 3:Java::JavaLang::IllegalArgumentException
This error will apear in the logs when trying to start ArchivesSpace after daylight savings time.
Reference: http://lyralists.lyrasis.org/mailman/htdig/archivesspace_users_group/2019-March/006652.html

Basically, the indexer user (or other user, but the indexer is the most likely culprit at 2am)
is being updated in the database with a time that does not exist due to daylight savings time.

To verify query for users with an `mtime` in the most recent
[daylight savings date](https://en.wikipedia.org/wiki/Daylight_saving_time_in_the_United_States):

```
SELECT * FROM user WHERE (user_mtime >= '2022-03-13 02:00:00' and user_mtime <= '2022-03-13 03:00:00') OR (system_mtime >= '2022-03-13 02:00:00' and system_mtime <= '2022-03-13 03:00:00');
```

The record that should come back is the `search_indexer` user, but could come back with others too.

To resolve update that user (and/or others) with a valid timestamp:

```
UPDATE user set user_mtime = NOW(), system_mtime=NOW() where username='search_indexer';
```

## Non-MSU Users of this Repository

This repository should be fairly re-usable by other institutions. The things you will need to look
to customize from this for yourselves if you fork this repository are:

* Updating references to `findingaids*.lib.msu.edu` to your URL
* Updating the `proxy-config/default.conf` with IP restrictions relavent to you
* Updating the `plugins` directory with your plugin/theme. It should be ok to leave the existing
  plugin that is already there, it just won't be used if your `config.rb` does not enable it.
* Reviewing the `deploy` script, specifically the `clone_plugins` function that determines
  which plugins to install if you want to add/remove from that list. You could optionally change
  the default value for `TARGET_DIR` defined at the top of the script if you want to avoid having
  to pass that as a parameter.
* Review the `sample.gitlab-ci.yml` as a starting point for making your own `.gitlab-ci.yml` file
  to deploy to your environment.

Please reach out if you have any questions or suggestions for improvement!

Email: [schanzme@msu.edu](mailto:schanzme@msu.edu)
