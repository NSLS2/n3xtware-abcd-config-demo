# Demo Configuration For N3XTware Ansible Based Configuration and Deployment (ABCD)

Configuration is currently accomplished on a per-host basis. The files herein are a substitute to the `host_vars` directories in Ansible.
The directories are named after the hosts that they are intended for, with YAML configuration according to the [ioc-deploy-roles n3xtware branch](https://github.com/nsls2/ioc-deploy-roles/tree/n3xtware).

## IOC Configuration Conventions

When deploying the IOCs for a host, the ansible role `DeployIOC` will iterate over the `iocs` list in the `ioc.yml` file in the host configuration directory.
Each IOC in the list should have a corresponsing configuration file in the directory with a dictionary with a single key of the IOC name (e.g., `xspress3.yml` should have a single dictionary with the key `xspress3`).
The value of that key should be a dictionary with keys specific to the IOC; for more information consult the [ioc-deploy-roles n3xtware branch](https://github.com/nsls2/ioc-deploy-roles/tree/n3xtware).

The `host.yml` file should contain the following keys, with examples below:

```yaml
    beamline: TLA
    hostname: DIRNAME
    softioc_user: softioc-TLA
    softioc_group: n2sn-instadmin
```

These variables will be loaded in place of `host_vars` in the `DeployIOC` role (e.g., through a combination of git clone and [`ansible.builtin.include_vars`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_vars_module.html)).
