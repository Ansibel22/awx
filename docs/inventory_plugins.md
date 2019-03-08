# Transition to Ansible Inventory Plugins
Inventory updates change from using scripts which are vendored as executable
python scripts in the AWX folder `awx/plugins/inventory` (taken originally from
Ansible folder `contrib/inventory`) to using dynamically-generated
YAML files which conform to the specifications of the `auto` inventory plugin
which are then parsed by their respective inventory plugin.

The major organizational change is that the inventory plugins are
part of the Ansible core distribution, whereas the same logic used to
be a part of AWX source.

## Prior Background for Transition

AWX used to maintain logic that parsed `.ini` inventory file contents,
in addition to interpreting the JSON output of scripts, re-calling with
the `--host` option in the case the `_meta.hostvars` key was not provided.

### Switch to Ansible Inventory

The CLI entry point `ansible-inventory` was introduced in Ansible 2.4.
In Tower 3.2, inventory imports began running this command
as an intermediary between the inventory and
the import's logic to save content to database. Using `ansible-inventory`
eliminates the need to maintain source-specific logic,
relying on Ansible's code instead. This also allows us to
count on a consistent data structure outputted from `ansible-inventory`.
There are many valid structures that a script can provide, but the output
from `ansible-inventory` will always be the same,
thus the AWX logic to parse the content is simplified.
This is why even scripts must be ran through the `ansible-inventory` CLI.

Along with this switchover, a backported version of
`ansible-inventory` was provided that supported Ansible versions 2.2 and 2.3.

### Removal of Backport

In AWX 3.0.0 (and Tower 3.5), the backport of `ansible-inventory`
was removed, and support for using custom virtual environments was added.
This set the minimum version of Ansible necessary to run _any_
inventory update to 2.4.

## Inventory Plugin Versioning

Beginning in Ansible 2.5, cloud sources in Ansible started migrating
away from "contrib" scripts (meaning they lived in the contrib folder)
to the inventory plugin model.

In AWX 4.0.0 (and Tower 3.5) inventory source types start to switchover
to plugins, provided that sufficient compatibility is in place for
the version of Ansible present in the custom virtualenv where the import
is being ran.

To see what version the plugin transition will happen, see
`awx/main/models/inventory.py` and look for the source name as a
subclass of `PluginFileInjector`, and there should be an `initial_version`
which is the first version that testing deemed to have sufficient parity
in the content its inventory plugin returns. For example, `openstack` will
begin using the inventory plugin in Ansible version 2.8, because the
openstack inventory plugin had an issue with sending logs to stdout which
was fixed in that version. If you run an openstack inventory update in
2.8 or lower, it will use the script.

### Sunsetting the scripts

Eventually, it is intended that all source types will have moved to
plugins. For any given source, after the `initial_version` for plugin use
is higher than the lowest supported Ansible version, the script can be
removed and the logic for script credential injection can also be removed.

For example, after AWX no longer supports Ansible 2.7, the script
`awx/plugins/openstack_inventory.py` will be removed.

## Changes to Expect in Imports

An effort was made to keep imports working in the exact same way after
the switchover. However, the inventory plugins are a fundamental rewrite
and many elements of default behavior has changed. Because of that,
a `compatibility_mode` toggle was added.

In a data migration, all existing cloud sources are switched over to
use `compatibility_mode`. New inventory sources will default to having
this off.

We recommend that you opt out of compatibility mode because this is more
future-proof, and also suggest that you set the `overwrite`
flag to help assure stale content is removed.

### Changes with Compatibility Mode Off

If no `group_by` entries are given, then no constructed groups will be
produced. That means no grouping by tags, regions, or similar attributes
unless the user adds these to the `group_by` listing. This is different
from prior behavior, where a blank `group_by` field would include all
possible groups.

The set of `hostvars` will be almost completely different, using new names
for data which is mostly the same content. You can see the jinja2 keyed_groups
construction used in compatibility mode to help get a sense of what
new names replace old names.

In many case, the host names will change. In many cases, accurate host
tracking will still be maintained via the host `instance_id`, but this
is not guaranteed.

Group names will be sanitized. That means that characters such as "-" will
be replaced by underscores "\_". In some cases, this means that a large
fraction of groups get renamed as you move from scripts to plugins.
Sanitizing group names allows referencing them in jinja2 in playbooks
without errors.

### Changes with Compatibility Mode On

Programatically-generated examples of inventory file syntax used in
updates (with dummy data) can be found in `awx/main/tests/data/inventory/scripts`,
these demonstrate the inventory file syntax used to restore old behavior
from the inventory scripts.

#### hostvar keys

More hostvars will appear. The inventory plugins name hostvars differently
that the contrib scripts did. To maintain backward compatibility,
the old names are added back where they have the same meaning as a
variable returned by the plugin. New names are not removed.

Caution: if you do not have `overwrite_vars` set
to True and you _downgrade_ the version of Ansible that an import runs in,
this will leave some stale hostvars.

Some hostvars will be lost, because of general deprecation needs.

 - ec2, see https://github.com/ansible/ansible/issues/52358
 - gce (see https://github.com/ansible/ansible/issues/51884)
   - `gce_uuid` this came from libcloud and isn't a true GCP field
     inventory plugins have moved away from libcloud

#### Host names

Host names might change, but tracking host identity via `instance_id`
will still be reliable.

The syntax of some hostvars, for some values, will change.

 - ec2
   - old: "ec2_block_devices": {"sda1": "vol-xxxxxx"}
   - new: "ec2_block_devices": {"/dev/sda1": "vol-xxxxxx"}
 - Azure
   - old: "tags": None
   - new: "tags": {}

## How do I write my own Inventory File?

If you do not want any of this compatibility-related functionality, then
you can add an SCM inventory source that points to your own file.
You can also apply a credential of a `managed_by_tower` type to that inventory
source that matches the cloud provider you are using, as long as that is
not `gce` or `openstack`.

All other sources provide _secrets_ via environment variables, so this
can be re-used without any problems for SCM-based inventory, and your
inventory file can be used securely to specify non-sensitive configuration
details such as the keyed_groups to provide, or hostvars to construct.

## Notes on Technical Implementation of Injectors

For an inventory source with a given value of the `source` field that is
of the standard cloud providers, a credential of the corresponding
credential type is required in most cases (exception being ec2 IAM roles).
This privileged credential is obtained by the method `get_cloud_credential`.

The `inputs` for this credential constitute one source of data for running
inventory updates. The following fields from the
`InventoryUpdate` model are also data sources, including:

 - `source_vars`
 - `source_regions`
 - `instance_filters`
 - `group_by`

The way these data are applied to the environment (including files and 
environment vars) is highly dependent on the specific cloud source.

With plugins, the inventory file may reference files that contain secrets
from the credential. With scripts, typically an environment variable
will reference a filename that contains a ConfigParser format file with
parameters for the update, and possibly including fields from the credential.

Caution: It is highly discouraged to put secrets from the credential into the
inventory file for the plugin. Right now there appears to be no need to do
this, and by using environment variables to specify secrets, this keeps
open the possibility of showing the inventory file contents to the user
as a latter enhancement.

Logic for setup for inventory updates using both plugins and scripts live
inventory injector class, specific to the source type.

Any credentials which are not source-specific will use the generic
injection logic which is also used in playbook runs.