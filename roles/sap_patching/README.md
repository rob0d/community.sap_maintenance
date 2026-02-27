
# Ansible Role sap_patching

This Ansible role automates the patching of different componentes of SAP NetWeaver ABAP / S/4HANA systems and HANA DB platform. It provides tasks for preparing the patching source files, applying the patches and stopping/starting the system as and when required. The role supports advanced restart strategies such as ABAP kernel bootstrap or Rolling Kernel Switch (RKS) for kernel updates.

## Overview

### Supported components

The following components are currently supported:

- ABAP Kernel [S/4 HANA 2023+ with kernel 7.93 tested, NW7.50+ should work]
- HANA Database client (both local and central installation)
- HANA Database platform edition [HANA 2.07-09 tested, HANA 2.04+ should work]
  - Full upgrade in "one go"
  - Upgrade split into prepare and upgrade steps
  - HANA System Replication enabled systems [tested with upto four systems in multi target configuration]
  - Two-node HA cluster with Pacemaker and HANA System Replication
- SAP Host Agent (both automated and manual update) [7.22+]
- SAP Web Dispatcher[7.93+ tested, 7.7+ should work]

### Patching steps

The following table summarises the steps which can/must be performed when patching each of the supported components.

| Component \ Step  | Prepare | Application stop | Apply | Application start | Kernel restart      | Post             |
|-------------------|---------|------------------|-------|-------------------|---------------------|------------------|
| ABAP kernel       | ✓       | Optional         | ✓     | No                | Full restart or RKS | Only with RKS    |
| HANA DB client    | ✓       | Required         | ✓     | Required          | N/A                 | N/A              |
| HANA DB platform  | ✓       | Optional         | ✓     | If stopped        | N/A                 | N/A              |
| Host Agent        | ✓       | No               | ✓     | No                | No                  | N/A              |
| Web Dispatcher    | ✓       | No               | ✓     | No                | Full restart        | No               |

As can be seen above, the patching process of each component consists of several steps (typicaly at least two):

- **Prepare:**
  This validates and prepares patch files. SAR files are read from a source directory and then extracted and archived in an appropriate directory in preparation foe application. For kernel this includes extraction of kernel patches in the correct order (see below).
- **Apply:**
  This executes the patching process. In some cases this step may require that the system is down (see the table above). Generally this step copies files prepared in the prepare step to the correct location, sets the permissions (if required) and executes other required task as per the component type.
- **Stop/Start/Restart:**
  This allows to stop/start the whole system. Kernel/Webdisp restart is a special step which deals with the intricacies of ABAP kernel patching (e.g. bootstrap, saproot.sh, etc).
- **Post:**
  Performs any post-patching actions. Currently only ABAP kernel Rolling Kernel Switch needs a specific post action.

**Refer to the relevant section for details on each supported component and its patching workflow.**

### Patching process

The patching process is controlled using list `sap_patching_execution_plan`. The steps in the execution plan and their order determines what  happens and when. For example to patch ABAP kernel and restart the system you can use the following execution plan:

```yaml
sap_patching_execution_plan:
  - abap_kernel_prepare
  - abap_kernel_apply
  - abap_kernel_restart
```

A more complex example includes the preparation of kernel, DB client and host agent files, upgrade of host agent, full stop of ABAP system, application of client libraries, application of a new kernel and start of ABAP system can look like this:

```yaml
sap_patching_execution_plan:
  - abap_kernel_prepare
  - hdb_client_prepare
  - host_agent_prepare
  - host_agent_apply_auto
  - abap_stop_system
  - hdb_client_apply
  - abap_kernel_apply
  - abap_kernel_restart
  - abap_start_system
```

The behaviour of each step can be controlled using task specific parameters described below.

## Prerequisites and Common Parameters

### Prerequisites

#### Source SAR files

SAR files must be downloaded and placed in the appropriate source directory or directories before running this role. You can obtain SAR files manually using the SAP Download Manager or the [community.sap_launchpad Ansible collection](https://github.com/sap-linuxlab/community.sap_launchpad).
Note that kernel source SAR directory must be currently separate from all other source SAR directories and can contain ONLY kernel related SAR and info files.

#### Shared vs Local patch directories

Both the source SAR files and extracted patches must be stored in directories accessible from the system which is being patched. You can choose between two approaches:

- **Shared directories e.g. NFS**: _This is the simplest and most efficient option_. The shared directories have to be present on all servers that belong to one SAP system. They can be different between SAP systems i.e. Dev, QA and Prod can have different NFS shares.
- **Local directories**: Each server AND Ansible controller must have their own local directory to store the source SAR files and extracted patches. This is controlled by setting `sap_patching_*_extracted_shared` parameters to `false`. Some components (e.g. HANA DB client and server) require a local directory on each application/database server where the patches have to be copied from the controller node. In this case use `sap_patching_*_extracted_local_path` to set where Ansible will store the extracted files locally.

### Common parameters

- `sap_patching_execution_plan`: As described above
- `sap_patching_sap_system_sid`: SAP System SID (e.g., "S4H")
- `sap_patching_hdb_system_sid`: SAP HANA DB platform SID (e.g., "HDB") - required only for HANA DB platform patching (not for HANA client)
- `sap_patching_wdp_system_sid`: SAP Web Dispatcher SID (e.g., "WDP") - required only for Web Dispatcher patching
- `sap_patching_sapcar_path`: Path to SAPCAR executable
- `sap_patching_sapcar_file_name`: SAPCAR executable filename

## Kernel Patching

The following steps are available:

- **Kernel Preparation:**
  - Step name `abap_kernel_prepare`.
  - Validates and locates kernel SAR files in the source directory.
  - NOTE: Due to the difficulties with _safely_ identifying kernel SAR files the source directory can contain ONLY the kernel SAR files (or nothing).
  - Handles duplicate detection and patch number extraction.
  - Supports both autodetect and manual kernel version selection.
- **Kernel Application:**
  - Step name `abap_kernel_apply`.
  - Validates kernel directories and files.
  - Copies kernel files to the global SAP executable directory.
- **ABAP Kernel Restart:**
  - Step name `abap_kernel_restart`.
  - Supports RKS, full system restart and bootstrap strategies.
  - Ensures a proper update of all special files, bootstrap, system restart and execution of saproot.sh (saproot.sh is not executed with RKS restart strategy, see below).
- **Post-Patching Permission Adjustment:**
  - Step name `abap_kernel_saproot`.
  - Executes `saproot.sh` to adjust permissions (icmbnd* and sapuxuserchk) after patching.
  - Explicit execution of this step is required only with RKS restart strategy (it is executed automatically with full restart and bootstrap).

### Variables for ABAP Kernel Patching

- `sap_patching_kernel_base`: Base directory for kernel SP directories and files.
- `sap_patching_kernel_source`: This is where all kernel SAR files need to be located during the prepare step. The files will be moved to `sap_patching_kernel_base\<kernel_version>` during the prepare step and this directory will be left empty. *NOTE: the source directory can contain ONLY the kernel SAR files (or nothing).
- `sap_patching_kernel_prepare_version`: (`autodetect` or a specific version).
What kernel version are we extracting from SAR files - set to 'autodetect' to use disp+work to determine the extracted kernel version and patch number. If for some reason you don't want (or can't) use autodetection, set this a specific kernel version e.g. 793 (format is `<NNN>` where N is a number). The kernel patch will be determined from the name of the SAR files.
- `sap_patching_kernel_prepare_appendto_version`: To be used when not patching dw or SAPEXE e.g. when adding extra patch like R3trans to existing kernel patch (format `<kernel_version>p<patch_number>`). See  Append-to mode below.
- `sap_patching_kernel_to_apply`: Either relative (to `sap_patching_kernel_base`) or absolute path to the kernel which will be applied. If not set, `sap_patching_kernel_to_apply_symlink` will be used. `abap_kernel_prepare` task will set this if it was executed and kernel was extracted. If you want to apply the latest extracted kernel, don't set this variable and make sure you run the prepare step once before running the apply step (this can be in two separate execution plans or together inside one execution plan).
- `sap_patching_kernel_extracted_shared`: (`true` or `false`) If true, the kernel source SAR files are expected to be present on the local host (e.g. NFS share). If false, the kernel source SAR files will be copied from the Ansible controller to the target host.
- `sap_patching_kernel_restart_strategy`: Restart strategy (`bootstrap`, `all`, `rks`)
  - `bootstrap` - Will restart only sapstartsrv and run sapcpe (use this when combining kernel with HANA client updates and want to restart only once).
  - `rks` - Rolling Kernel Switch. Make sure you understand the implications of this and the behaviour of the system. If required, adjust the SAP profile parameters accordinly.
  - `all` - Simple stop/start of the whole system (all running instances).

### Kernel Usage

See `sample-sap_patching_kernel.yml` and `sample-sap_patching_full.yml`.

### Preparation modes

The default (and recommended) mode is `autodetect`. This is set using `sap_patching_kernel_prepare_version`. This will take all SAR files from the source directory, perform checks and prepare a new shiny kernel to be applied.

However, if the source directory doesn't contain a full complement of SAR files two additional behaviours are possible:

- **Delta mode:** If the source directory contains only dw*sar files and NOT SAPEXE/SAPEXEDB files it means that this is not a full kernel. Therefore the extracted kernel will have suffix **-delta** (e.g., `k793p330-delta`). This indicates that this is not a full working kernel, just a delta over the latest full kernel. You may have a full kernel `k793p300` in a different directory. Therefore, to apply a full patch number 330 you may need to apply patch 300 and then run the patching process again to apply 330. This is cumbersome and prone to user errors, but there is a genuine use case, therefore this functionality is provided. The recommendation is to stick with full kernel every time patching is executed.
- **Append-to mode:** If there is a requirement to patch non-disp_work files e.g. R3trans or tp these can be appended to an existing prepared kernel. This operation will not change the kernel patch number (disp+work remains the same) but the kernel files have been updated. The source directory must contain some kernel patches and must not contain dw&ast;.sar or SAPEXE&ast;.sar (because this would change the kernel patch number). If you need to top-up the existing kernel set `sap_patching_kernel_prepare_appendto_version`.

### FAQ

- **My system is running on multiple platforms and I need to patch them both**
- Separate all folders and create a structure which takes both platforms into the account. Run abap_kernel_prepare twice each time with different settings. Run abap_kernel_apply twice as well and make sure that it is executed once on a server for each platform (e.g. once on linuxx86_64 and once on linuxs390x).

## HANA DB Client Patching

The following steps are available:

- **Client Preparation:**
  - Step name `hdb_client_prepare`.
  - Validates and prepares the HANA DB client installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Client Application:**
  - Step name `hdb_client_apply`.
  - Installs or updates the HANA DB client on target systems.
  - Both local and shared client installations are supported.

### Variables for HANA DB client patching

- `sap_patching_hdb_client_base`: Base HDB client directory where the HDB client SAR file are stored and extraction directories are created.
- `sap_patching_hdb_client_source`: Where the source SAR files are located (used during the prepare step).
- `sap_patching_hdb_client_to_apply`: Either relative (to `sap_patching_hdb_client_base`) or absolute path to the HDB client which will be applied. If not set, `sap_patching_hdb_client_to_apply_symlink` will be used. `hdb_client_prepare` task will set this if it was executed and HDB client was extracted. If you want to apply the latest extracted HDB client, don't set this variable and make sure you run the prepare step once before running the apply step (this can be in two separate execution plans or together inside one execution plan).
- `sap_patching_hdb_client_install_path`: If using individual client installation - installation path of the HDB client (used for --path option of hdbinst). Note this is mutually exclusive with `sap_patching_hdb_client_shared_path`.
- `sap_patching_hdb_client_shared_path`: If using shared client installation - path to the shared directory  (used for --sapmnt option of hdbinst). Note this is mutually exclusive with `sap_patching_hdb_client_install_path`.
- `sap_patching_hdb_client_extracted_shared`: If true, the extracted HDB client source files are expected to be present on the local host (e.g. NFS share) following the execution of the prepare step. If false, the extracted HDB client source files will be copied from the Ansible controller to the target host.
- `sap_patching_hdb_client_extracted_local_path`: Where to store a local copy of extracted HDB client files as they must be present locally in order to run hdbinst.

### HANA DB client Usage

See `sample-sap_patching_others.yml` and `sample-sap_patching_full.yml`.

## HANA DB Server Patching

The following steps are available:

- **Server Preparation:**
  - Step name `hdb_server_prepare`.
  - Validates and prepares the HANA DB server installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Server Application Preparation:**
  - Step name `sap_patching_hdb_server_update_prepare`.
  - Performs all update preparation steps upto the downtime phase.
  - Optional step which reduces the downtime duration.
- **Server Application:**
  - Step name `hdb_server_apply`.
  - Updates the HANA DB server on target systems.
  - If update preparation was executed beforehand, it will automatically continue with the downtime phase of the update.
  - If update preparation was NOT executed, it will automatically perform the full update/upgrade process.

### Variables for HANA DB server patching

- `sap_patching_hdb_server_base`: Base HDB server directory where the HDB server SAR file are stored and extraction directories are created.
- `sap_patching_hdb_server_source`: Where the source SAR files are located (used during the prepare step).
- `sap_patching_hdb_server_to_apply`: Either relative (to `sap_patching_hdb_server_base`) or absolute path to the HDB server which will be applied. If not set, `sap_patching_hdb_server_to_apply_symlink` will be used. `hdb_server_prepare` task will set this if it was executed and HDB server was extracted. If you want to apply the latest extracted HDB server, don't set this variable and make sure you run the prepare step once before running the apply step (this can be in two separate execution plans or together inside one execution plan).
- `sap_patching_hdb_server_install_path`: Installation path of the HDB server (used to check the existing installation). Unless you have a weird non-standard installation, there is no need to set this variable.
- `sap_patching_hdb_server_cf_system_user`: If set the specified user will be used to perform the update. READ the documentation at [SAP Help here](https://help.sap.com/docs/SAP_HANA_PLATFORM/2c1988d620e04368aa4103bf26f17727/df3de8c31cef45c0847d2804b97604ea.html) to understand the implications of using this option (XSA update does not work with this option for example).
- `sap_patching_hdb_server_cf_system_user_password`: Password for the HANA user used to perform the update (this is the SYSTEM user unless defined otherwise in sap_patching_hdb_server_cf_system_user).
- `sap_patching_hdb_server_activate_system_user`: If set to true, the system user (SYSTEM) will be activated prior to patching. If set to false or not set, no action will be taken.
- `sap_patching_hdb_server_activate_admin_userstore_key`: HANA admin user store key (hdbuserstore) to use when activating the SYSTEM user. If not set, sap_patching_hdb_server_activate_admin_user and sap_patching_hdb_server_activate_admin_password will be used instead.
- `sap_patching_hdb_server_activate_admin_user`: HANA admin user to use when activating the SYSTEM user.
- `sap_patching_hdb_server_activate_admin_password`: Password of the HANA admin user used when activating the SYSTEM user.
- `sap_patching_hdb_server_is_clustered`: If the database is clustered (currently only Redhat/Suse with Pacemaker are supported), set this to true.
- `sap_patching_hdb_server_clu_hdb_resource_name`: Name of the cluster resource for HDB server (used to relocate and put the resource in maintenance mode during patching). If SAP LinuxLab role `sap_ha_pacemaker_cluster` was used to setup the cluster AND HANA instance number is 00, this doesn't need to be setup as the default value matches the naming convention used by the role. The default value is: "cln_SAPHana_{{ sap_patching_hdb_system_sid | upper }}_HDB00".
- `sap_patching_hdb_server_extracted_shared`: If true, the extracted HDB server source files are expected to be present on the local host (e.g. NFS share) following the execution of the prepare step. If false, the extracted HDB server source files will be copied from the Ansible controller to the target host.
- `sap_patching_hdb_server_extracted_local_path`: Where to store a local copy of extracted HDB server files as they must be present locally in order to run hdblcm.
- `sap_patching_hdb_server_cf_*`: Variables which will be injected into the HDBLCM configuration file (see below).

### HDBLCM Configuration file

Dynamically generated configuration file is used to control how HDBLCM is behaving. This is very similar to the behaviour of ```community.sap_install.sap_hana_install role```. The configuration file is created in folder `ansible_templates/<github_hash>/<hostname>_hdblcm_configfile.*`.  Where:

- `<github_hash>` is basically a unique version of the HDBLCM. This changes with each HANA DB Server patch and it will force generation of a new config file even if one existed for a previous version of HDBLCM.
- `<hostname>_hdblcm_configfile.*` represent several files which start with the HANA DB server hostname. These are used to generate the config file (suffix .cfg) which is then passed to HDBLCM during execution.

The configuration file is generated only when it doesn't exist. This is inline with ```community.sap_install.sap_hana_install role``` behaviour. This also allows users to define their own additional variables in format `sap_patching_hdb_server_cf_<configfile_variable>` which will replace the default values in HBBLCM configuration file. This can be used to alter the behaviour of HDBLCM during the update/upgrade. For more details see the comments in `tasks/hana/hdblcm_configfile.yml`.

### HANA DB Server Usage

See `sample-sap_patching_hdb_server.yml` and `sample-sap_patching_full.yml`.

## Host Agent Patching

The following steps are available:

- **Host Agent Preparation:**
  - Step name `host_agent_prepare`.
  - Validates and prepares the installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **Host Agent Application:**
  - Step name `host_agent_apply_auto` for automatic agent update (See N1473974) or `host_agent_apply_manual` for manual update on each server.
  - Installs or updates the host agent on target systems.
  - Both local and shared agent installations are supported.

### Variables for host agent patching

- `sap_patching_host_agent_base`: Base host agent directory where the host agent SAR file are stored and extraction directories are created.
- `sap_patching_host_agent_source`: Where the source SAR files are located (used during the prepare step).
- `sap_patching_host_agent_to_apply`: Either relative (to `sap_patching_host_agent_base`) or absolute path to the host agent which will be applied. If not set, `sap_patching_host_agent_to_apply_symlink` will be used. `host_agent_prepare` task will set this if it was executed and host agent was extracted. If you want to apply the latest extracted host agent, don't set this variable and make sure you run the prepare step once before running the apply step (this can be in two separate execution plans or together inside one execution plan).
- `sap_patching_host_agent_autoupdate_path`: If host agent autoupdate is configured (<https://me.sap.com/notes/1473974>) this is the path where it's looking for updates (parameter DIR_NEW in host_profile)
- `sap_patching_host_agent_extracted_shared`: If true, the extracted host agent source files are expected to be present on the local host (e.g. NFS share) following the execution of the prepare step. If false, the extracted host agent source files will be copied from the Ansible controller to the target host.
- `sap_patching_host_agent_extracted_local_path`: Where to store a local copy of extracted host agent files as they must be present locally in order to run saphostexec. This only applies to manual agent update.

## SAP Web Dispatcher Patching

The following steps are available:

- **WebDisp Preparation:**
  - Step name `webdisp_prepare`.
  - Validates and prepares the installation files.
  - Ensures the correct version and patch level are selected for deployment.
- **WebDisp Application:**
  - Step name `webdisp_apply`.
  - Installs or updates the host agent on target systems.
  - Both local and shared agent installations are supported.
- **WebDisp Restart:**
  - Step name `webdisp_restart`.
  - Ensures proper update of all special files, bootstrap, system restart and execution of saproot.sh.

### Variables for WebDisp patching

- `sap_patching_webdisp_base`: Base web disp directory where the web disp SAR file are stored and extraction directories are created
- `sap_patching_webdisp_source`: Where the source SAR files are located (used during the prepare step)
- `sap_patching_webdisp_to_apply`: Either relative (to `sap_patching_webdisp_base`) or absolute path to the web disp which will be applied. If not set, `sap_patching_webdisp_to_apply_symlink` will be used. `webdisp_prepare` task will set this if it was executed and web disp was extracted. If you want to apply the latest extracted web disp, don't set this variable and make sure you run the prepare step once before running the apply step (this can be in two separate execution plans or together inside one execution plan).
- `sap_patching_webdisp_extracted_shared`: If true, the extracted web disp files are expected to be present on the local host (e.g. NFS share) following the execution of the prepare step. If false, the extracted web disp files will be copied from the Ansible controller to the target host

### WebDisp Usage

See `sample-sap_patching_others.yml` and `sample-sap_patching_full.yml`.

## License

See LICENSE file for details.

## Maintainers

Rob Dobozy at sap
