# Tutorial: Using the Ansible Based Configuration and Deployment (ABCD) for Sim Devices

In this tutorial we will be using the ABCD to configure and deploy the following:

1. EPICS IOCs for a simulated area detector and simulator motor
2. Engineering screens to monitor and control the area detector and motor [Phoebus][]
3. Pythonic abstrations of these devices and an Ipython profile to run them [Bluesky][]

## Prerequisites

## Step 1: Clone the ABCD Configurations Repository

```bash
git clone https://github.com/nsls2/n3xtware-abcd-config-demo.git
```

### Purpose of the configuration repository (?)
This configuration provides a mimic of the configuration structure for `ioc-deploy-roles`. 
This is to support a demonstration that deploys IOCs to a host on a LAN, and enable users to develop and modify configuration on an isolated network.

### Structure of the configuration repository

Configuration is currently accomplished on a per-host basis. The files herein are a substitute to the `host_vars` directories in Ansible.
The directories are named after the hosts that they are intended for, with YAML configuration according to the [ioc-deploy-roles n3xtware branch][].
For example, to set up `host_vars` for this demo, we will use `<hostname>/host.yml`. For a `localhost` deployment, it would be under `localhost/host.yml`.

### IOC Configuration Conventions

When deploying the IOCs for a host, the ansible role `DeployIOC` will iterate over the `iocs` list in the `ioc.yml` file in the host configuration directory.
Each IOC in the list should have a corresponsing dictionary with key of the IOC name, which points to the configuration of the IOC. Then, the IOC configuration should be included in that same directory for the same host.
The value of that key should be a dictionary with keys specific to the IOC; for more information consult the [ioc-deploy-roles n3xtware branch][].

The `host.yml` file should contain the following keys, with examples below:

```yaml
beamline: TLA
hostname: DIRNAME
softioc_user: softioc-TLA
softioc_group: n2sn-instadmin
```

These variables will be loaded in place of `host_vars` in the `DeployIOC` role (e.g., through a combination of git clone and [`ansible.builtin.include_vars`][]).

## Step 2: Configure a simulated motor

A sensible file-tree for the IOC configuration would look like this:

```plaintext
n3xtware-abcd-config-demo
├── localhost
│   ├── ioc.yml
│   ├── host.yml
│   ├── motorsims.yml
│   ├── areadetectors.yml
```
Where we can group the configurations of individual IOCs based on the `type` of IOC into a single configuration file.

However, for this demo, we would suggest using individual files for each IOC configuration to avoid merge conflicts. A downstream PR could refactor these many related IOCs into a single file.

```plaintext
n3xtware-abcd-config-demo
├── localhost
│   ├── ioc.yml
│   ├── host.yml
│   ├── motor1.yml
│   ├── motor2.yml
│   ├── motor3.yml
...
```

The motorsim role will construct an EPICS IOC and Bluesky Ophyd class. The IOC `type` for the ansible deployment is `MotorSim`. The class will use default component names of `ax{i}` if none are provided; however a `class_name` and `instance_name` are required for the Ophyd class generation.
Currently only traditional ophyd sync is supported with ophyd-async support planned.

Please follow the example configuration and assemble your own motor sim IOC config in a distinct file in the `localhost` directory. Push these changes to a branch, and open a PR to the `main` branch. The PR will be reviewed and merged by the proctoring team.

```yml
mc-sim1:
    type: "MotorSim"  # Specifies that this is a motor sim IOC instance. Leave as is
    environment:
        SYS: "XF:31ID1-ES" # First portion of prefix. Should match `P` attribute in substitutions
        CT_PREFIX: "XF:31ID1-CT{SimMC:01}" # The admin controls PV prefix. (used for utils/ioc admin functions)
        ENGINEER: "J. Wlodek" # Your name
        PORT: "MTR1" # Asyn port ID. Must be same as PORT in substitutions. Typically leave as is
        NUM_AXES: 8 # Number of axes
    substitutions:
        motorSim:
        templates:
            SimMotor:
            filepath: "$(TOP)/db/SimMotor.template"
            #         Base Prefix | Motor name | Asyn Port | Axis # | Description | Units
            pattern: ["P",          "M",         "PORT",     "ADDR",  "DESC",       "EGU"]
            instances:
                - ["XF:31ID1-ES", "{SimMC:01-Ax:1}", "MTR1", "0", "Sim Axis 1", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:2}", "MTR1", "1", "Sim Axis 2", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:3}", "MTR1", "2", "Sim Axis 3", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:4}", "MTR1", "3", "Sim Axis 4", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:5}", "MTR1", "4", "Sim Axis 5", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:6}", "MTR1", "5", "Sim Axis 6", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:7}", "MTR1", "6", "Sim Axis 7", "mm"]
                - ["XF:31ID1-ES", "{SimMC:01-Ax:8}", "MTR1", "7", "Sim Axis 8", "mm"]
    ophyd:
        class_name: "MotorSim"
        instance_name: "my_motor_sim"
        components:
        - ax1
        - ax2
        - ax3
        - ax4
        - ax5
        - ax6
        - ax7
        - ax8
```

## Step 3: Configure a simulated area detector

Similar to the steps you took for adding a simulated motor, you will need to add a simulated area detector.  The IOC `type` for the ansible deployment is `ADSimDetector`.  The area detector role will construct an EPICS IOC and Bluesky Ophyd class. The class will include plugins as specified in the `ophyd.components` list. Currently available components are:

- `image`
- `stats`
- `transform`
- `process`
- `roi`
- `tiff`
- `hdf5`

If using `tiff` or `hdf5` additional configuration is required in the `.ophyd` dictionary in th IOC configuration file. These are all passed to the filestore mixin.

- `ophyd.write_path_template`
- `ophyd.read_path_template`
- `ophyd.root`

It is possibhle to use generic names for the classes where available, and to use the IOC name for the instance name.
Duplicate classes will require the same configuration to be included exactly. See below.

Please follow the example configuration and assemble your own motor sim IOC config in a distinct file in the `localhost` directory. Push these changes to a branch, and open a PR to the `main` branch. The PR will be reviewed and merged by the proctoring team.

```yml
cam-sim1: # Name of your IOC, should match name of yml file and your 
  type: "ADSimDetector" # Type of IOC - should always stay ADSimDetector
  environment:
      PREFIX: "XF:31ID1-ES{SIM-Cam:1}" # EPICS PV Prefix. See BL PV naming conventions
      ENGINEER: "J. Wlodek" # Your Name

      # Dimensions of the simulated image in x and y
      XSIZE: "1024"
      YSIZE: "1024"

  ophyd:
      class_name: "SimDetectorTIFF"
      instance_name: "cam-sim1"
      components:
      - "image"
      - "stats"
      - "roi"
      - "process"
      - "transform"
      - "tiff"
      write_path_template: "/tmp/%Y/%m/%d/"
      read_path_template: "/tmp/%Y/%m/%d/"
      root: "/tmp"

  cam-sim2: # Name of second IOC
  type: "ADSimDetector" # Type of IOC - should always stay ADSimDetector
  environment:
      PREFIX: "XF:31ID1-ES{SIM-Cam:2}" # EPICS PV Prefix. See BL PV naming conventions
      ENGINEER: "J. Wlodek" # Your Name

      # Dimensions of the simulated image in x and y
      XSIZE: "1024"
      YSIZE: "1024"

  ophyd:
      class_name: "SimDetectorTIFF" # Because this is an existing class, the configuration must be the same.
      instance_name: "cam-sim2" # New instance name
      # Since the class name is the same, the components and other configuration must be the same
      components:
      - "image"
      - "stats"
      - "roi"
      - "process"
      - "transform"
      - "tiff"
      write_path_template: "/tmp/%Y/%m/%d/"
      read_path_template: "/tmp/%Y/%m/%d/"
      root: "/tmp"
```

## Step 4: Deploy the IOCs

For the tutorial session, we will be performing a deployment on an IOC server by running the ansible roles as root on the server.
Doing this, we avoid some of the complexities of Ansible deployments that are accomplished in our facility wide deployment (e.g., networking, inventory, ansible-vault, privilege escalation).
For details on the NSLS-II ansible deployment and Ansible Automation Platform for managing our inventories, see the [NSLS-II Docs].

Begin by logging into the IOC server and navigating to the `ioc-deploy-roles` repository: `cd /opt/n3xtware/ioc-deploy-roles/`.
Log in by ssh as root with the IP and password provided by the proctors, or logging in directly to the workstation if available.
From here, we will write a single playbook to deploy our newly configured motor and detector IOCs, and their [Bluesky] configuration. We will deploy [Phoebus] screens for all IOCs at once in a single playbook.

```yaml
---
# ioc-deploy-roles/my-test-playbook.yml
- name: Deploy Motor and Detector IOCs
  hosts: localhost
  vars:
    - RetrieveABCDConfig_repo: https://github.com/nsls2/n3xtware-abcd-config-demo.git
    - RetrieveABCDConfig_key_file: id_ed25519_n3xtware_abcd_config_demo
    - iocs_to_deploy:
        - MOTOR1 # CHANGE ME
        - DET1 # CHANGE ME
    - ansible_python_interpreter: /usr/bin/python3
    - DeployIpythonProfile_ipython_dir: /opt/bluesky

  tasks:
    - name: Ensure job is running against a single host
      ansible.builtin.fail:
        msg: Refusing to run on more than one host
      when: ansible_play_hosts | length > 1

    - name: Retrieve ABCD Config
      ansible.builtin.include_role:
        name: RetrieveABCDConifg

    - name: Deploy EPICS common files
      ansible.builtin.include_role:
        name: DeployCommon
      vars:
        - host_epics_intf: 192.168.1.165 # Forced for demo off loopback

    - name: Deploy Motor and Detector IOCs
      ansible.builtin.include_role:
        name: DeployIOC
      loop: "{{ iocs_to_deploy }}"
      loop_control:
        loop_var: ioc_name
        index_var: ioc_install_loop_index
      vars:
        - _post_deploy_step: "Install and Enable and Start"

    - name: Deploy Bluesky Ipython Profile
      ansible.builtin.include_role:
        name: DeployIpythonProfile
      vars:
        - beamline_acronym: "tst"
        - operator_account: "operator"
        - DeployIpythonProfile_specific_iocs: "{{ iocs_to_deploy }}"
        - RetrieveABCDConfig_ioc_server: localhost
```

Run the playbook with the following command:

```bash
ansible-playbook my-test-playbook.yml --limit localhost --vault-password-file /etc/ansible/vault_pass.txt
```

If you are a DSSI staff member, you can login to `runansible[1-3].nsls2.bnl.local` and clone the `ioc-deploy-roles` repository. From there you can run the above playbooks directed toward hosts in which you are an administrator with root access. In production these tools exist as templates in AAP, and can be accessed by the GUI.

## Step 5: Deploy Phoebus Screens

Phoebus screens are deployed all at once, since the main screen is organized according to all IOCs on a beamline. The `PhoebusAutogen` role will automate the generation of Phoebus screens for N3XTWare using the specified beamline configuration in the ABCD config repository.
This role retrieves the necessary repositories, installs required Python packages, and runs the `phoebus_autogen` module to generate the screens.

```yaml
---
- name: Run Phoebus Autogen
hosts: localhost
tasks:
  - name: PhoebusAutogen
    ansible.builtin.include_role:
      name: PhoebusAutogen
    vars:
      - PhoebusAutogen_beamline: tst
      - PhoebusAutogen_retrieve_autogen_repos_enabled: true
      - PhoebusAutogen_key_file: id_ed25519_n3xtware_phoebus_autogen_demo
```

## Step 6: Run your 'beamline'

With these steps completed you can now login to the server as the "operator".
In practice, this would be a dedicated account for running the beamline on a workstation distinct from the IOC server; however, for simplicity we will accomplish this as a different user on a single machine.

We can accomplish this via ssh using X forwarding, or by logging in directly to the workstation if available.

```bash
ssh -X operator@IP
```

From here you can run the Bluesky run engine and interact with the IOC through the Bluesky interface.

```bash
ipython --profile=n3xtware
```

```python
RE(scan([my_motor_sim], my_motor_sim.ax1, 0, 1, 1))
```

You can also inspect and interact with the devices using the autoconfigured engineering screens.

```bash
run-phoebus
```

## Rough Setup Notes for Proctoring Local Demo (for future reference)

- Provision host with RHEL 8 to use as `localhost` in config
- Install epel: sudo `subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms`
- Install epics and manage-iocs from [nsls2 rpm repo][]
- Install ansible with dnf
- Set up root user with pwd `n3xtware-admin`
- Set up operator user with pwd `n3xtware`
- Add group `n2sn-inststaff-tst` and add operator to it: `groupadd n2sn-inststaff-tst`; `usermod -aG n2sn-inststaff-tst operator`
- As `root` clone the [ioc-deploy-roles n3xtware branch][] into `/opt/n3xtware/`
- Make a branch of ioc-deploy-roles from the `n3xtware` branch, and replace deploy keys encrypted with new `ansible-vault` password stored in `/etc/ansible/vault_pass.txt` for the following. This keeps our and vault pass unique and separate from the demo.
  - [demo config repo][] in RetrieveABCDConfig role
  - [phoebus autogen repo][] in PhoebusAutogen role
- Create a mongo service on the ioc server
- Create a tiled profile for tst to direct connect to mongo
- Setup phoebus core using the `install_phoebus_all.yml` playbook in the `ansible-epics-tools` repository. This will require some downloading and creativity. (FWIW, Phil let `install_phoebus_all.yml` fail to get java, `install_phoebus.yml` to get phoebus, then manually found and added [phoebus preferences repo][] to `/opt/css/preferences`)
- Necessary to use a local interface for EPICS so that broadcast works. The loopback (127.0.0.1/8) causes problems.
- 

### Mongo service

```systemd
# container-mongo.service
# autogenerated by Podman 4.9.4-rhel

[Unit]
Description=Podman container-mongo.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
    --cidfile=%t/%n.ctr-id \
    --cgroups=no-conmon \
    --rm \
    --sdnotify=conmon \
    --replace \
    -d \
    --name mongo \
    --volume mongo:/data/db \
    --publish 27017:27017 docker.io/library/mongo:latest
ExecStop=/usr/bin/podman stop \
    --ignore -t 10 \
    --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm \
    -f \
    --ignore -t 10 \
    --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

### Tiled Profile

```yml
# /etc/tiled/profiles/tst.yml 
tst:
direct:
    authentication:
    allow_anonymous_access: true
    trees:
    - tree: databroker.mongo_normalized:Tree.from_uri
        path: /
        args:
        uri: mongodb://localhost:27017/mad-bluesky-documents
```

[Bluesky]: https://blueskyproject.io/
[Phoebus]: https://controlssoftware.sns.ornl.gov/css_phoebus/
[NSLS-II Docs]: https://docs.nsls2.bnl.gov
[ioc-deploy-roles n3xtware branch]: https://github.com/nsls2/ioc-deploy-roles/tree/n3xtware
[`ansible.builtin.include_vars`]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_vars_module.html
[nsls2 rpm repo]: https://www-dev.nsls2.bnl.gov/nsls2/rhel/$releasever/$basearch/
[demo config repo]: https://github.com/nsls2/n3xtware-abcd-config-demo.git
[phoebus autogen repo]: https://github.com/NSLS2/n3xtware-phoebus-autogen.git
[phoebus preferences repo]: https://code.nsls2.bnl.gov/cs-studio-nsls2/preferences
