# Panorama Object Mover — Ansible Edition

Move PAN-OS objects between Panorama scopes (**device group ↔ device group**, **shared ↔ device group**, **shared ↔ shared**) using pure Ansible.
This mirrors the Python version's behavior: create in destination, then delete from source, with collision handling.

## New features
- **Dry-run** mode: previews actions without changing Panorama.
- **Dependency checker**: for `address-group` and `service-group`, verifies member objects exist in destination scope or in `shared` before moving (optional).
- **Optional commit**: trigger a basic Panorama commit after moves.

## What it does
- Reads `objects.csv` (one row per object to move)
- For each row:
  - Fetches the full XML `<entry>` from the source scope
  - Optionally **skips** or **overwrites** if the destination already has an object with the same name
  - **Dry-run**: shows planned actions; otherwise `set` the `<entry>` in the destination container path, then `delete` the source entry
  - **Dependency check** (optional): for groups, ensures members exist in destination or shared before proceeding
  - **Commit** (optional): commits Panorama config at the end

> This playbook does **not** push to firewalls; it only commits Panorama (if enabled).

## Layout
```
panorama-object-mover-ansible/
├─ move_objects.yml
├─ files/
│  └─ objects.csv
├─ inventories/
│  └─ default/
│     ├─ inventory.yml
│     └─ group_vars/
│        └─ all.yml
└─ collections/
   └─ requirements.yml
```

## Requirements
- Ansible 2.13+
- Collections:
  ```bash
  ansible-galaxy collection install -r collections/requirements.yml
  ```

## Configure
Edit `inventories/default/group_vars/all.yml`:
```yaml
panorama_ip: "192.168.1.10"
api_key: "YOUR_API_KEY"

# skip | overwrite
collision_policy: "skip"

# Path to the CSV file
objects_csv: "files/objects.csv"

# New options
dry_run: false              # true = preview only
check_dependencies: true    # only applies to address-group / service-group
commit_changes: false       # true = commit Panorama at the end
```

## Input CSV
`files/objects.csv` must have this header:
```
object_name,object_type,src_scope,dst_scope
```
Example:
```
object_name,object_type,src_scope,dst_scope
test-addr-1,address,Branch1,shared
test-svc-1,service,shared,Branch2
vip-db,address,DC-Core,Branch2
web-ports,service-group,shared,shared
prod-tags,address-group,Branch1,DC-Core
```

## Run
```bash
# Default inventory and vars
ansible-playbook -i inventories/default/inventory.yml move_objects.yml

# Example: dry-run only
ansible-playbook -i inventories/default/inventory.yml move_objects.yml -e dry_run=true

# Example: enforce dependency checks and commit afterwards
ansible-playbook -i inventories/default/inventory.yml move_objects.yml -e check_dependencies=true -e commit_changes=true
```

## Notes
- Uses `paloaltonetworks.panos.panos_xpath` to `set`/`delete`, and `uri` to `get` the source `<entry>` XML.
- For **shared** scope, container path is `/config/shared/<type>`.
- For device groups, container path is `/config/devices/entry[@name='localhost.localdomain']/device-group/entry[@name='<DG>']/<type>`.
- Moving groups (address-group / service-group) may require moving their members first; enable `check_dependencies` to enforce.


## Logs
Every run writes a timestamped CSV under `./logs/`, e.g. `moves_YYYYmmdd_HHMMSS.csv` with headers:
```
timestamp,object_name,object_type,src_scope,dst_scope,status,message
```
Possible `status` values:
- `planned` — dry-run preview only
- `skipped` — destination already has the object and `collision_policy=skip`
- `moved` — created in destination and deleted from source
- `error` — the move failed (see `message` for details)
```bash
tail -f logs/moves_*.csv
```
