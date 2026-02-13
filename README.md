# community.sap_maintenance Ansible Collection

![Ansible Lint](https://github.com/rob0d/community.sap_maintenance/actions/workflows/ansible-lint.yml/badge.svg?branch=main)

## Description

This Ansible Collection provides automation for SAP system maintenance tasks on supported Linux operating systems. It is designed to simplify and standardize SAP maintenance operations, including patching and upgrades.

Included roles cover a range of maintenance tasks:
- Patching of SAP S/4 HANA components
  - ABAP Kernel [S/4 HANA 2023+ with kernel 7.93 tested, NW7.50+ should work]
  - HANA Database client (both local and central installation)
  - HANA Database platform edition [HANA 2.07-09 tested, HANA 2.04+ should work]
    - Full upgrade in "one go"
    - Upgrade split into prepare and upgrade steps
    - HANA System Replication enabled systems [tested with upto four systems in multi target configuration]
    - Two-node HA cluster with Pacemaker and HANA System Replication
  - SAP Host Agent (both automated and manual update) [7.22+]
  - SAP Web Dispatcher[7.93+ tested, 7.7+ should work]

- Automation of any maintenance using SUM (including upgrades, migrations, DMO, DO-DMO)
  - THIS IS NOT IMPLEMENTED YET (Development in progress)


## Requirements

### Control Nodes
Operating system:
- Any operating system with required Python and Ansible versions.

Python: 3.6 or higher

Ansible: 9.9.x

Ansible-core: 2.16.x


### Managed Nodes
Operating system:
- SUSE Linux Enterprise Server for SAP applications 15 SP5+ (SLE4SAP)
- Red Hat Enterprise Linux for SAP Solutions 8.x 9.x (RHEL4SAP)

**NOTE: Operating system needs to have access to required package repositories either directly or via subscription registration.**

Python: 3.6 or higher

## Installation Instructions

### Installation
Install this collection with Ansible Galaxy command:
```console
ansible-galaxy collection install community.sap_maintenance
```

Optionally you can include the collection in a requirements.yml file and include it together with other collections using: `ansible-galaxy collection install -r requirements.yml`
Requirements file needs to be maintained in the following format:
```yaml
collections:
  - name: community.sap_maintenance
```

### Upgrade
Installed Ansible Collection will not be upgraded automatically when Ansible package is upgraded.

To upgrade the collection to the latest available version, run the following command:
```console
ansible-galaxy collection install community.sap_maintenance --upgrade
```

You can also install a specific version of the collection, when you encounter issues with the latest version. Please report these issues in the affected Role repository if that happens.
Example of downgrading collection to version 1.0.0:
```
ansible-galaxy collection install community.sap_maintenance:==1.0.0
```

See [Installing collections](https://docs.ansible.com/ansible/latest/collections_guide/collections_installing.html) for more details on installation methods.

## Use Cases

### Example Scenarios
- Preparation of operating system for SAP maintenance
- Maintenance of SAP system files and directories
- Execution of SAP system checks and health reports
- Application of SAP patches and updates
- Configuration and validation of SAP system settings

### Ansible Roles
All included roles can be executed independently or as part of your own playbooks.

| Name | Summary |
| :--- | :--- |
| [sap_patching](https://github.com/rob0d/community.sap_maintenance/tree/main/roles/sap_patching) | Patching of different SAP components |
| [slt_sum](https://github.com/rob0d/community.sap_maintenance/tree/main/roles/slt_sum) | System upgrades and migrations |

## Testing
This Ansible Collection was tested across different Operating Systems and SAP maintenance scenarios. You can find examples of some of them below.

Operating systems:
- SUSE Linux Enterprise Server for SAP applications 15 SP5+ (SLE4SAP)
- Red Hat Enterprise Linux for SAP Solutions 8.x 9.x (RHEL4SAP)

Deployment scenarios:
- Common SAP maintenance tasks, including patching, system checks, and configuration updates

SAP Products:
- SAP S/4HANA AnyPremise (1809, 1909, 2020, 2021, 2022, 2023, 2025) single, distributed or clustered installations
- SAP Business Suite (ECC) on HANA
- SAP HANA 2.0 (SPS04+) with setup as Scale-Up, Scale-Out, High Availability

**NOTE: It is not possible to test every Operating System and SAP Product combination with every release. Testing is regularly done for common maintenance scenarios.**

## Contributing
For information on how to contribute, please see our [contribution guidelines](https://sap-linuxlab.github.io/initiative_contributions/).


## Contributors
We welcome contributions to this collection. For a list of all contributors and information on how you can get involved, please see our [CONTRIBUTORS document](./CONTRIBUTORS.md).

## Support
You can report any issues using [Issues](https://github.com/sap-linuxlab/community.sap_install/issues) section.


## Release Notes and Roadmap
You can find the release notes of this collection in [Changelog file](https://github.com/sap-linuxlab/community.sap_install/blob/main/CHANGELOG.rst)


## Further Information

### Variable Precedence Rules
Please follow [Ansible Precedence guidelines](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) on how to pass variables when using this collection.

### Getting Started
More information on how to execute Ansible playbooks is in [Getting started guide](https://github.com/sap-linuxlab/community.sap_install/blob/main/docs/getting_started/README.md).


## License
[LGPL 2](https://github.com/rob0d/community.sap_maintenance/blob/main/LICENSE)
