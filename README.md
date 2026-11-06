# galaxias.wp_safe_update

Ansible collection that updates WordPress sites with WP-CLI behind a **Proxmox VE
snapshot**: if a site is broken after the update, the VM (or container) is rolled back.

For each host (one VM or container that can host several sites):

1. **Preflight**, per site
   - WordPress is installed.
   - **Baseline smoke test** from the controller. A site that is already broken is skipped, so the update doesn't get the blame.
   - **Integrity check** (`wp core/plugin verify-checksums`). A mismatch *before* the update can mean a compromise: the site is skipped, so the evidence isn't overwritten.
   - Lists the pending core, plugin and theme updates. With nothing to do, the host stops here, without a snapshot.
2. **Database dump** of every site to update (`wp db export`, gzipped, only the latest one is kept).
3. **Snapshot** of the VM through the Proxmox API (`wp-safe-update-<UTC timestamp>`). Older `wp-safe-update-*` snapshots are removed once the new one exists, so the latest one stays available until the next run.
4. **Update**, per site, in maintenance mode: core (major or minor only), database, plugins, themes, translations, your extra commands (`wc update`...), object cache flush.
5. **Verify**, per site
   - WordPress loads with all plugins and the theme (`wp eval`), which catches PHP fatal errors.
   - The smoke tests pass again.
   - The checksums still match.
6. **On any failure**: roll back to the snapshot, start the VM, wait for it, smoke test the sites again, then fail with a report.

```
TASK [galaxias.wp_safe_update.wp_update : Report]
    wp_update__summary:
        host: web1
        snapshot: wp-safe-update-20261028-155314
        rolled_back: false
        sites:
            blog:
                status: updated
                core: [6.8.2 -> 6.8.3]
                plugins:
                -   name: classic-editor
                    version: 1.6.3
                    update_version: 1.7.0
```

## What a rollback means

Read this before you rely on it:

- **The whole VM is rolled back.** If one site breaks, every site on that VM goes back to its state before the run, and so does anything else running on the VM.
- **Writes made during the run are lost**: orders, comments, form submissions made between the snapshot and the rollback. Maintenance mode (on by default) keeps visitors out while each site is updated, not during the verification.
- **A database on another machine is not in the snapshot.** The `wp db export` dumps (`/var/backups/wp-safe-update` on the host) are your way back; restoring them is up to you.
- **On ZFS storage**, Proxmox can only roll back to the most recent snapshot. The snapshot taken by the run is always the latest, unless someone takes another one during the run.

## Requirements

- On the controller:
  - ansible-core >= 2.16
  - `community.proxmox` >= 1.5.0, installed automatically as a dependency
  - `proxmoxer` and `requests` for the Python running Ansible
- On the hosts:
  - WP-CLI
  - `runuser` (util-linux)
  - `mysqldump`/`mariadb-dump` for the database dumps
- A Proxmox VE API token, see below.

## Install

```bash
ansible-galaxy collection install galaxias.wp_safe_update
```

or in `requirements.yml`:

```yaml
collections:
  - name: galaxias.wp_safe_update
    version: ">=0.1.0"
```

Every option is documented in `ansible-doc -t role galaxias.wp_safe_update.wp_update`.

## Usage

Describe the sites in your inventory (see [examples/inventory.yml](examples/inventory.yml)
and every option in [roles/wp_update/defaults/main.yml](roles/wp_update/defaults/main.yml)):

```yaml
wordpress:
  hosts:
    web1:
      wp_update_proxmox_vmid: 101   # not needed with the community.proxmox inventory plugin
      wp_update_sites:
        - name: blog
          path: /var/www/blog
          user: blog                # owner of the files, WP-CLI runs as this user
```

Then call the role from a playbook of your own:

```yaml
# wp-safe-update.yml
- name: Safely update WordPress sites
  hosts: wordpress
  gather_facts: false   # not needed by the role
  become: true          # WP-CLI runs as each site's owner through runuser
  serial: 1             # one VM at a time
  max_fail_percentage: 0  # stop at the first failed VM
  roles:
    - role: galaxias.wp_safe_update.wp_update
```

```bash
# What would be updated, without snapshot or changes
ansible-playbook wp-safe-update.yml -i inventory.yml -K -e wp_update_dry_run=true

# Update
ansible-playbook wp-safe-update.yml -i inventory.yml -K
```

`max_fail_percentage: 0` stops the run as soon as one VM fails: a plugin update that broke
the sites of one VM is likely to break the next one too. Raise `serial` to update several
VMs at once.

### Proxmox API token

Minimal privileges: read the VM, snapshot, roll back, start.

```bash
pveum role add WPUpdate --privs "VM.Audit,VM.Snapshot,VM.Snapshot.Rollback,VM.PowerMgmt"
pveum user add ansible@pve
pveum user token add ansible@pve wp-safe-update --privsep 1
# With privilege separation, the token gets the intersection of its own ACLs and the user's
pveum acl modify /vms --users ansible@pve --roles WPUpdate
pveum acl modify /vms --tokens 'ansible@pve!wp-safe-update' --roles WPUpdate
```

Restrict `/vms` to a pool (`/pool/<name>`) if you like. With `wp_update_proxmox_vmstate: true`
(RAM included in the snapshot), the token also needs `Datastore.AllocateSpace` on the storage.

```yaml
wp_update_proxmox_api_host: pve.example.com
wp_update_proxmox_api_user: ansible@pve
wp_update_proxmox_api_token_id: wp-safe-update
wp_update_proxmox_api_token_secret: "{{ vault_wp_update_proxmox_token_secret }}"
```

Empty settings fall back to the `PROXMOX_HOST`, `PROXMOX_USER`, `PROXMOX_TOKEN_ID`,
`PROXMOX_TOKEN_SECRET`... environment variables.

### Without Proxmox

`wp_update_snapshot_provider: none` runs the same checks, updates and report without a
snapshot: a broken site is reported, not rolled back. Useful to try the collection out, or
for hosts outside Proxmox. Other providers (libvirt, cloud volumes...) only need a
`snapshot.yml` and a `rollback.yml` in `roles/wp_update/tasks/providers/<name>/`;
contributions are welcome.

## Notes

- Updates run as each site's owner (`runuser`), never as root: WP-CLI can't write files
  the site couldn't write itself.
- `wp_update_load_plugins: true` (default) lists and updates plugins and themes with plugins
  loaded, so premium plugins with their own updater are updated too. Core updates and
  checks always run with `--skip-plugins --skip-themes`. Must-use plugins are always loaded.
- Plugins not hosted on wordpress.org have no checksums: they are skipped by the integrity
  check. Locally modified plugins go in the site's `checksum_exclude_plugins`.
- Smoke tests run from the controller, with a cache-busting query string. Add checks
  for the pages that matter (cart, checkout, API) with `smoke_checks`.
- Only plugins and themes listed in preflight are updated, so the report is exactly what
  was done. Translations are always refreshed.
- With `opcache.validate_timestamps=0`, list your PHP-FPM services in `wp_update_reload_services`.

## Development

```bash
git clone https://github.com/galaxias-fr/ansible-collection-wp-safe-update.git
# Use it from another project without reinstalling after each change
mkdir -p collections/ansible_collections/galaxias
ln -s /path/to/ansible-collection-wp-safe-update collections/ansible_collections/galaxias/wp_safe_update
# ansible.cfg: collections_path = ./collections:~/.ansible/collections
```

To try changes without Proxmox, point the role at a test VM or container with an older
WordPress install and `wp_update_snapshot_provider: none`: it covers everything but the
snapshot itself.

Checks run by the CI:

```bash
ansible-lint
# from collections/ansible_collections/galaxias/wp_safe_update
ansible-test sanity --docker
```
