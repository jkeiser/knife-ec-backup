# Knife EC Backup
[![Build status](https://badge.buildkite.com/4bc85427aab66accafbd7abb2932b9dd7f9208162c5be33488.svg?branch=master)](https://buildkite.com/chef-oss/chef-knife-ec-backup-master-verify)
[![Gem Version](https://badge.fury.io/rb/knife-ec-backup.svg)](https://badge.fury.io/rb/knife-ec-backup)

**Umbrella Project**: [Knife](https://github.com/chef/chef-oss-practices/blob/master/projects/knife.md)

**Project State**: [Active](https://github.com/chef/chef-oss-practices/blob/master/repo-management/repo-states.md#active)

**Issues [Response Time Maximum](https://github.com/chef/chef-oss-practices/blob/master/repo-management/repo-states.md)**: 14 days

**Pull Request [Response Time Maximum](https://github.com/chef/chef-oss-practices/blob/master/repo-management/repo-states.md)**: 14 days

## Description

knife-ec-backup can backup and restore the data in an Enterprise Chef
Server installation, preserving the data in an intermediate, editable
text format.  It is similar to the `knife download` and `knife upload`
commands and uses the same underlying libraries, but also includes
workarounds for objects not yet supported by those tools and various
Server API deficiencies.  The long-run goal is to improve `knife
download`, `knife upload` and the Chef Infra Server API and deprecate this
tool.

## Requirements

This knife plugin requires Chef Infra Client 11.8+.

### Server Support

This plugin currently supports Enterprise Chef 11 and Chef Infra Server 12+.
Support for the beta key rotation features is provided via the
`--with-keys-sql` flag, but users of this feature should note that
this may change once the Chef Infra Server supports an API-based export of
the key data.

## Installation

### Chef Infra Server Install (Recommended)

This gem is installed with chef-server-core 12.0.0 and newer.

For Private Chef 11 (or Enterprise Chef 11) you'll need to download and build
locally to get the correct dependencies, either with `git clone` or by
downloading the .zip file. Once unpacked, run:

```
/opt/opscode/embedded/bin/gem build knife-ec-backup.gemspec
/opt/opscode/embedded/bin/gem install knife-ec-backup*gem --no-ri --no-rdoc -V
```

#### Note on installing with existing development tools:

The latest versions of knife-ec-backup require gems with native
extensions, thus you must install a standard build toolchain.  To
install knife-ec-backup without installing libpq development headers
on your system, try the following:

   /opt/opscode/embedded/bin/gem install knife-ec-backup -- --with-pg-config=/opt/opscode/embedded/postgresql/9.2/bin/pg_config

This uses the libpq headers that are included in the Chef Infra Server
package installed in `/opt/opscode`.

### Chef Workstation Install (Unsupported)

On systems other than the Chef Infra Server, installation of this gem is not
tested or supported. However, if you attempt to do so you will need the
postgresql libraries installed.

For example, on macOS:

```
brew install libpq
gem install knife-ec-backup -- --with-pg-config=/usr/local/Cellar/libpq/9.2/bin/pg_config
```

The current location of pg_config can be determined with `brew info libpq`.

## Running tests

```
$ bundle install
$ bundle exec rspec spec/
```

If bundle install fails on the pg gem and the note above does not work for you, try:

```
$ brew install postgres (if not present)
$ ARCHFLAGS="-arch x86_64" gem install pg
$ bundle exec rspec spec/
```

## Build from source

Clone the git repository and run the following from inside:

    gem build knife-ec-backup.gemspec
    gem install knife-ec-backup*gem

## Configuration

### Permissions

Note that most users in an EC installation lack the permissions to pull all of the data from all organizations and other users.
This plugin **REQUIRES THE PIVOTAL KEY AND WEBUI KEY** from the Chef Infra Server.
It is recommended that you run this from a frontend Enterprise Chef Infra Server, you can use --user and --key to pass the pivotal information along.

## Subcommands

### Common Options

The following options are supported across all subcommands:

  * `--sql-host`:
    The hostname of the Chef Infra Server's postgresql server. (default: localhost)

  * `--sql-port`:
    The postgresql listening port on the Chef Infra Server. (default: 5432)

  * `--sql-db`:
    The postgresql Chef Infra Server database name. (default: opscode_chef)
    Specify 'automate-cs-oc-erchef' when using Automate Chef Infra Server API

  * `--sql-user`:
    The username of postgresql user with access to the opscode_chef
    database. (default: autoconfigured from
    /etc/opscode/chef-server-running.json)

  * `--sql-password`:
    The password for the sql-user.  (default: autoconfigured from /etc/opscode/chef-server-running.json)

  * `--purge`:
    Whether to sync deletions from backup source to restore destination. (default: false)

  * `--dry-run`:
    Report what actions would be taken without performing any. (default: false)

### knife ec backup DEST_DIR (options)

*Path*: If you have chef-client installed as well, you may need to invoke this as `/opt/opscode/embedded/bin/knife ec backup backup`

*Options*

  * `--concurrency THREAD_COUNT`:
    The maximum number of concurrent requests to make to the Chef
    Server. (default: 10)

  * `--webui-key`:
    Used to set the path to the WebUI Key (default: /etc/opscode/webui_priv.pem)
    skip any auto-configured options (default: false)

  * `--with-user-sql`:
    Whether to backup/restore user data directly from the database.  This
    requires access to the listening postgresql port on the Chef
    Server.  This is required to correctly handle user passwords and
    to ensure user-specific association groups are not duplicated.

  * `--with-key-sql`: Whether to backup/restore key data directly
    from the database.  This requires access to the listening
    postgresql port on the Chef Infra Server.  This is required to correctly
    handle keys in Chef Infra Servers with multikey support. This option
    will only work on `restore` if it was also used during the
    `backup`.

  * `--skip-useracl`:
    Skip download/restore of the user ACLs.  User ACLs are the
    permissions that actors have *on other global users*.  These are
    not the ACLs that control what permissions users have on various
    Chef objects.

  * `--skip-version-check`:
    Skip Chef Infra Server version check. This will also skip any auto-configured options (default: false)

  * `--only-org ORG`:
    Only donwload/restore objects in the named organization. Global
    objects such as users will still be downloaded/restored.

Creates a repository of an entire Enterprise Chef / Private Chef server.

The format of the repository is based on the `knife-essentials` (`knife download`) format and looks like this:

    users
      <name>.json
    user_acls
      <name>.json
    organizations
      <orgname>
        acls
          <type>
            <name>.json
        clients
          <name>.json
        containers
          <name>.json
        cookbooks
          <name>-<version>
        data_bags
          <bag name>
            <item name>
        environments
          <name>.json
        groups
          <name>.json
        nodes
          <name>.json
        roles
          <name>.json
        org.json
        members.json
        invitations.json

This compares very closely with the "knife download /" from an OSC server:

    clients
      <name>.json
    cookbooks
      <name>-<version>
    data_bags
      <bag name>
        <item name>
    environments
      <name>.json
    nodes
      <name>.json
    roles
      <name>.json
    users
      <name>.json>

### knife ec restore DEST_DIR (options)

Restores all data from the specified DEST_DIR to an Enterprise Chef /
Private Chef server. DEST_DIR should be a backup directory created by
`knife ec backup`

*Options*

  * `--webui-key`:
    Used to set the path to the WebUI Key (default: /etc/opscode/webui_priv.pem)

  * `--overwrite-pivotal`:
    Whether to overwrite pivotal's key.  Once this is done, future
    requests will fail until you fix the private key (default: false)

  * `--skip-users`:
    Skip the restore of global users.  This may cause organization
    uploading to fail if the necessary users do not exist on the Chef
    Server.

  * `--concurrency THREAD_COUNT`:
    The maximum number of concurrent requests to make to the Chef
    Server. (default: 10)

  * `--skip-version-check`:
    Skip Chef Infra Server version check. This will
    also skip any auto-configured options (default: false)

  * `--[no-]skip-user-ids`:
    Reuses user ids from the restore destination when updating existing
    users to avoid database conflicts (default: true)

  * `--with-user-sql`:
    Whether to backup/restore user data directly from the database.  This
    requires access to the listening postgresql port on the Chef
    Server.  This is required to correctly handle user passwords and
    to ensure user-specific association groups are not
    duplicated. This option will only work on `restore` if it was also
    used during the `backup`.

  * `--with-key-sql`: Whether to backup/restore key data directly
    from the database.  This requires access to the listening
    postgresql port on the Chef Infra Server.  This is required to correctly
    handle keys in Chef Infra Servers with multikey support. This option
    will only work on `restore` if it was also used during the
    `backup`.

  * `--skip-useracl`:
    Skip download/restore of the user ACLs.  User ACLs are the
    permissions that actors have *on other global users*.  These are
    not the ACLs that control what permissions users have on various
    Chef objects.

  * `--only-org ORG`:
    Only download/restore objects in the named organization. Global
    objects such as users will still be downloaded/restored.

### knife ec key export [FILENAME]

Create a json representation of the users table from the Chef Infra Server
database.  If no argument is given, the name of the backup is
`key_dump.json`.

Please note, most users should use `knife ec backup` with the
`--with-user-sql` option rather than this command.

### knife ec key import [FILENAME]

Import a json representation of the users table from FILENAME to the
the Chef Infra Server database.  If no argument is given, the filename is
assumed to be `key_dump.json`.

Please note, most user should use `knife ec restore` with the
`--with-user-sql` option rather than this command.

## Known Bugs

- knife-ec-backup cannot be installed in the embedded gemset of Chef
  Server 12.  This will be resolved in a future Chef Infra Server release.

- `knife ec restore` can fail to restore cookbooks, failing with an
  internal server error. A common cause of this problem is a
  concurrency bug in Chef Infra Server. Setting `--concurrency 1` can often
  work around the issue.

- `knife ec restore` can fail if the pool of pre-created organizations
  can not keep up with the newly created organizations.  This can
  typically be resolved simply be restarting the restore.  To avoid
  this error for backups with large number of organizations, try
  setting (in /etc/opscode/private-chef.rb):

        opscode_org_creator['ready_org_depth']

  to the number of organizations in your backup and waiting for the
  pool to fill before running `knife ec restore`
