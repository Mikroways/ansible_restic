# Ansible Role: mikroways.restic

This role was inspired by the excelent article from
[Jack Henschel, Restic Backups with Systemd and Prometheus exporter](https://blog.cubieserver.de/2021/restic-backups-with-systemd-and-prometheus-exporter/).
I've also found an inspiration from other existing roles as: [arsillio.restic](https://github.com/arillso/ansible.restic)
but evolving from cron to systemd was my objective, so I've decided to write
this new role as a composition of ideas.

> This documentation needs to be updated with changes introduced since 2.0.x

## Description

This role installs [Restic](https://github.com/restic/restic) and configures as
many backup tasks to make backupsC as defined, each with its corresponding
restic repo.

So the fact is that you will be using this role to:

* Install restic
* Setup restic repos, named repo1, repo2, repo3
* Setup restic backup scripts to backup data from your system into some of the
  previously defined repos. Each of this backup scripts can then be used in
  different ways:
  * Manually
  * From a systemd timer

> Using systemd timer is more like using cronjobs. The main difference is the
> potencial systemd provides as ordering services, for example to mount a
> automount a filesystem previous to runing a backup script, drift randomly a
> scheduled task so not all tasks run at the same time (RandomizedDelaySec), and
> many other features that makes this a decoupled solution. Moreover, with
> systemd we can manage faiures, so when a backup failed we can manage what to
> do!

### Backup Scripts

This role will create as many backup scripts as you define. For each defined
backup, you will get three scripts:
* One with restic credentials
* Other with restic backup script and eventually a repository initialization
* The last one will prune your backups

This scripts will be stored in a configurable scripts directory:
`restic_script_dir`.

## Requirements

* bzip2

## Role Variables

| Name                            | Default         | Description                                                                 |
| ------------------------------- | ----------------| --------------------------------------------------------------------------- |
| `restic_url`                    | `undefined`     | The URL to download restic from. Use this variable to overwrite the default |
| `restic_version`                | `'0.12.0'`      | The version of Restic to install                                            |
| `restic_download_path`          | `'/opt/restic'` | Download location for the restic binary                                     |
| `restic_install_path`           | `'/usr/bin'`    | Install location for the restic binary                                      |
| `restic_script_dir`             | `'{{ restic_download_path }}/restic'` | Location of the generated backup scripts              |
| `restic_repos`                  | `{}`            | A dictionary of repositories where snapshots are stored                     |
| `restic_backups`                | `{}` (or `[]`)  | A list of dictionaries specifying the files and directories to be backed up |
| `restic_create_systemd_timer`   | `false`         | Configure systemd timer or not                                              |
| `restic_dir_owner`              | `{{ ansible_user }}` | User id to configure restic script folder                              |
| `restic_dir_group`              | `{{ ansible_user }}` | Group id to configure restic script folder                             |
| `restic_access_template`        | `restic_access.j2`   | Template to load used restic variables in backup scripts               |
| `restic_backup_template`        | `restic_backup.j2`   | Template script to make backups                                        |
| `restic_prune_template`         | `restic_prune.j2`    | Template script ro prune backups                                       |
| `restic_systemd_timer_template` | `systemd_timer.unit.j2` | Template for systemd timer unit                                     |
| `restic_systemd_service_template` | `systemd_service.unit.j2` | Template for systemd restic backup service called by timer      |
| `restic_systemd_failure_service_template` | `systemd_failure.unit.j2` | Template for systemd on error handling                  |
| `restic_systemd_failure_service_name` | `restic-backup-failure@.service` | Name of systemd service to handle errors             |
| `restic_prometheus_exporter_enabled` | `false` ||
| `restic_prometheus_exporter_template`| `restic_prometheus_exporter.j2` ||
| `restic_prometheus_exporter_script` | `{{ restic_script_dir }}/restic_prometheus_exporter.sh` ||
| `restic_prometheus_exporter_metrics_basedir` | `/var/lib/node-exporter` ||
| `restic_failure_template` | `restic_failure_unit.j2` ||
| `restic_failure_script` | `{{restic_script_dir }}/unit-failure` ||
| `restic_failure_mail_to` | `root` ||
| `restic_non_root_setup` | `false` | Installs Restic as a specific user with read-only rights. (See [restic docs](https://restic.readthedocs.io/en/stable/080_examples.html#backing-up-your-system-without-running-restic-as-root))|
| `restic_non_root_setup_user` | `restic` ||
| `restic_gomaxprocs` | not defined | By default, restic [uses all available CPU cores](https://restic.readthedocs.io/en/stable/047_tuning_backup_parameters.html#cpu-usage). Use this variable to limit the number of used CPU cores.   |

### Repos

Restic stores data in repositories. You have to specify at least one repository
to be able to use this role. A repository can be local or remote (see the
official [documentation](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)).

> **Using an SFTP repository**
>
> For using an SFTP backend, the user needs passwordless access to the host.
> Please make sure to distribute ssh keys accordingly, as this is outside of
> the scope of this role.

Available variables:

| Name                    | Required | Description                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------- |:--------:| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `location`              | yes      | The location of the Backend. Currently, [Local](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#local), [SFTP](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#sftp), [S3](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#amazon-s3) and [B2](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#backblaze-b2) are supported |
| `password`              | yes      | The password used to secure this repository                                                                                                                                                                                                                                                                                                                                                                        |
| `aws_access_key`        | no       | The access key for the S3 backend                                                                                                                                                                                                                                                                                                                                                                                  |
| `aws_secret_access_key` | no       | The secret access key for the S3 backend                                                                                                                                                                                                                                                                                                                                                                           |
| `aws_default_region`    | no       | The desired region for the S3 backend                                                                                                                                                                                                                                                                                                                                                                              |
| `b2_account_id`         | no       | The account ID for Backblaze B2 backend                                                                                                                                                                                                                                                                                                                                                                            |
| `b2_account_key`        | no       | The account key for Backblaze B2 backend                                                                                                                                                                                                                                                                                                                                                                           |
Repo will be initialized if it is not done.

Example:

```yaml
restic_repos:
  local:
    location: /srv/restic-repo
    password: securepassword1
  remote:
    location: sftp:user@host:/srv/restic-repo
    password: securepassword2
```

### Backups

A backup specifies a directory or file to be backuped. A backup is written to a
Repository defined in `restic_repos`.

Available variables:

| Name               | Required (Default)            | Description                                                                                                                                                                  |
| ------------------ |:-----------------------------:| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`             | yes                           | The name of this backup. Used together with pruning and scheduling and needs to be unique.                                                                                   |
| `repo`             | yes                           | The name of the repository to backup to.                                                                                                                                     |
| `src`              | yes                           | The source directory or file                                                                                                                                                 |
| `stdin`            | no                            | Is this backup created from a [stdin](https://restic.readthedocs.io/en/stable/040_backup.html#reading-data-from-stdin)?                                                      |
| `stdin_cmd`        | no (yes if `stdin` == `true`) | The command to produce the stdin.                                                                                                                                            |
| `stdin_filename`   | no                            | The filename used in the repository.                                                                                                                                         |
| `tags`             | no                            | Array of default tags                                                                                                                                                        |
| `keep_last`        | no                            | If set, only keeps the last n snapshots.                                                                                                                                     |
| `keep_hourly`      | no                            | If set, only keeps the last n hourly snapshots.                                                                                                                              |
| `keep_daily`       | no                            | If set, only keeps the last n daily snapshots.                                                                                                                               |
| `keep_weekly `     | no                            | If set, only keeps the last n weekly snapshots.                                                                                                                              |
| `keep_monthly`     | no                            | If set, only keeps the last n monthly snapshots.                                                                                                                             |
| `keep_yearly `     | no                            | If set, only keeps the last n yearly snapshots.                                                                                                                              |
| `keep_within`      | no                            | If set, only keeps snapshots in this time period.                                                                                                                            |
| `keep_tag`         | no                            | If set, keep snapshots with this tags. Make sure to specify a list.                                                                                                          |
| `prune`            | no (`false`)                  | If `true`, the `restic forget` command in the script has the [`--prune` option](https://restic.readthedocs.io/en/stable/060_forget.html#removing-backup-snapshots) appended. |
| `schedules`        | no (`[]`)                  | Array of calendar event expressions as [explained here](https://www.freedesktop.org/software/systemd/man/systemd.time.html#) |
| `exclude`          | no (`{}`)                     | Allows you to specify files to exclude. See [Exclude](#exclude) for reference.                                                                                               |

Example:

```yaml
restic_backups:
  data:
    name: data
    repo: remote
    src: /path/to/data
    schedules:
            -  "*-*-* 02,16:45:00"
```

#### Exclude

the `exclude` key on a backup allows you to specify multiple files to exclude or
files to look for filenames to be excluded. You can specify the following keys:

```yaml
exclude:
    exclude_cache: true
    exclude:
        - /path/to/file
    iexclude:
        - /path/to/file
    exclude_file:
        - /path/to/file
    exclude_if_present:
        - /path/to/file
```

Please refer to the use of the specific keys to the
[documentation](https://restic.readthedocs.io/en/latest/040_backup.html#excluding-files).

## Full backup without root

As explained in the Restic official docs, the tool can be [used without root](https://restic.readthedocs.io/en/stable/080_examples.html#backing-up-your-system-without-running-restic-as-root).
To have it installed as the docs are describing, set the desired user for Restic, e.g. `restic`:

```yml
restic_non_root_setup: true
restic_non_root_setup_user: restic
restic_dir_owner: restic
restic_dir_group: restic
```

## Dependencies

none

## Example Playbook

```yml
- hosts: all
  roles:
    - role: mikroways.restic
      become: true
```

## Author

- Christian Rodriguez

## License

This project is under the MIT License. See the [LICENSE](https://sbaerlo.ch/licence) file for the full license text.

## Copyright

(c) 2022, Mikroways
