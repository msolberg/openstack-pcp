# Monitoring RHEL-OSP with Performance Co-Pilot

This is a short guide for operators looking to leverage Performance Co-Pilot in the monitoring of their Red Hat OpenStack Platform deployments. Performance Co-Pilot is a suite of tools, services, and libraries for monitoring, visualizing, storing, and analyzing system-level performance measurements included in the Red Hat Enterprise Linux operating system. It is relatively easy to deploy and has several web-based tools available to graph system performance. This guide covers the most basic deployment and configuration of PCP for monitoring OpenStack clouds - there is much more information in the formal documentation:

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/setting-up-pcp_monitoring-and-managing-system-status-and-performance

## Installing PCP on Collector Hosts

Each host in the OpenStack deployment will have PCP installed to collect and log performance metrics. Red Hat provides a package ```pcp-zeroconf``` which will install and configure PCP will basic settings for metrics collection. In addition to installing the package and starting the services, a couple of configuration changes will need to be made to the Collector Hosts so that they expose metrics to the Monitoring Host.

First, the firewall will need to be configured to allow incoming TCP connections on port 44321. Each Role in the OpenStack Platform deployment will require a separate configuration to enable the firewall change. 

### Director
The director should have the following configuration added to ```/home/stack/hieradata.yaml```:

```
tripleo::firewall::firewall_rules:
  '44321 PCP Incoming':
    dport: 44321
    proto: tcp
    destination: 0.0.0.0/0
    action: accept
    state: ['NEW','ESTABLISHED','RELATED']
```

If the deployment does not currently use a hieradata override, the following will need to be added to the ```undercloud.conf```:

```
# Path to hieradata override file. If set, the file will be copied
# under /etc/puppet/hieradata and set as the first file in the hiera
hieradata_override = /home/stack/hieradata.yaml
```

To update the configuration, run an undercloud deployment:

```
# openstack undercloud install
```

### Controller, Compute, and other roles

The Controllers will also need the firewall configuration added the ControllerExtraConfig:

```
parameter_defaults:
  ControllerExtraConfig:
    tripleo::firewall::firewall_rules:
      '44321 PCP Incoming':
        dport: 44321
        proto: tcp
        destination: 0.0.0.0/0
        action: accept
        state: ['NEW','ESTABLISHED','RELATED']
```
Similarly, the Compute roles and other roles will also need the firewall rule added to their ExtraConfig parameters:
```
parameter_defaults:
  ComputeExtraConfig:
    tripleo::firewall::firewall_rules:
      '44321 PCP Incoming':
        dport: 44321
        proto: tcp
        destination: 0.0.0.0/0
        action: accept
        state: ['NEW','ESTABLISHED','RELATED']
```

Once these changes are put in place, the should be pushed out to the systems with an Overcloud deployment:

```
# openstack overcloud deploy
```

To avoid a deployment of the overcloud, these changes can also temporarily be pushed out with a command similar to the following:

```
# iptables -I INPUT 5 -p tcp -m multiport --dports 44321 -m state --state NEW,RELATED,ESTABLISHED -m comment --comment "44321 PCP Incoming ipv4" -j ACCEPT
```

### Installing the packages and configuration on all roles

An Ansible playbook is provided [here](playbooks/install_pcp.yaml) which will install and configure PCP on the Collector Hosts in the environment. It performs the following tasks:

1) Install the pcp-zeroconf package
2) Configure pmcd to listen on the Control Plane interface
3) Configure SELinux to allow pmcd to attach to the 44321 port.
4) Enable and start the pmcd and pmlogger services.

## Installing the Monitoring Host

A single host will be used to collect performance statistics from the OpenStack nodes, aggregate them, and then present them via a set of web interfaces. This host can either be RHEL 7 or RHEL 8. Official instructions for setting up the Collector on RHEL 8 are available [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/logging-performance-data-with-pmlogger_monitoring-and-managing-system-status-and-performance#setting-up-the-central-server-to-collect-data_logging-performance-data-with-pmlogger). Note that the configuration used in this guide will generate roughly 100M of performance logs per host per day.

### Installing Packages

The Performance Co-Pilot packages are included in the base repositories for RHEL 7 and RHEL 8. The web application package (```pcp-webjs```) is included in the ```rhel-7-server-optional-rpms``` repository for RHEL 7. To install the PCP packages on RHEL 7, run the following command:

```
# yum -y install pcp pcp-system-tools pcp-webjs pcp-webapi
```




