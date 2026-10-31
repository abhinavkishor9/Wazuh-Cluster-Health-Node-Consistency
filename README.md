# Wazuh Cluster Health and Node Consistency

## Overview

This lab examines Wazuh Manager health, cluster configuration, node visibility, and synchronization status in a Rocky Linux-based Wazuh deployment.

The assessment began with a standalone Wazuh Manager configuration in which clustering was disabled. Cluster-control commands, configuration inspection, service checks, network-listener inspection, and log analysis were used to understand the deployment's operational state.

A single-node master configuration was also attempted to explore Wazuh cluster functionality. Although the modified configuration passed XML syntax validation, the Manager failed to restart. The original configuration was subsequently restored, and the Manager service returned to an active state.

The exercise demonstrates why cluster health cannot be established from service status or configuration settings alone. Node membership, communication, synchronization, and the actual deployment topology must also be considered.

## Lab Objectives

- Establish the Wazuh deployment topology and component version baseline.
- Inspect the Manager's cluster configuration and identify its configured role.
- Understand the purpose of master and worker nodes.
- Examine cluster-control output and node synchronization information.
- Review Wazuh Manager service status and individual daemon states.
- Inspect relevant network listeners, including the cluster communication port.
- Identify operational warnings and errors in the Manager logs.
- Observe the impact of a cluster configuration change on Manager startup.
- Restore the previous configuration and verify service recovery.
- Document the limitations of cluster validation in a single-Manager environment.

## Environment

| Component | Details |
|---|---|
| Virtualization | VMware |
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Architecture | x86-64 |
| Network Interface | ens160 |
| Wazuh VM IP | 192.168.203.129 |
| Wazuh Manager | 4.13.1 |
| Wazuh Indexer | 4.13.1 |
| Wazuh Dashboard | 4.13.1 |
| Deployment | Single Wazuh VM |

The Wazuh control utility reported version `v4.13.1` and revision `rc1`.

## Assessment Areas

### 1. Cluster Configuration

The existing `/var/ossec/etc/ossec.conf` file was inspected to identify the cluster name, node name, node type, configured node address, communication port, bind address, and cluster-enabled setting.

The original configuration showed:

- Cluster name: `wazuh`
- Node name: `node01`
- Node type: `master`
- Cluster port: `1516`
- Bind address: `0.0.0.0`
- Cluster disabled: `yes`

### 2. Cluster Status and Node Visibility

The cluster-control utility was used to inspect cluster membership and synchronization information.

During the exercise, a captured cluster listing reported one master node:

- Node: `node01`
- Role: `master`
- Version: `4.13.1`
- Address: `192.168.203.129`

The corresponding cluster information output reported zero completed synchronizations for connected nodes. This did not demonstrate master-worker synchronization because the environment contained only one Wazuh VM.

### 3. Service and Network Health

The Wazuh Manager service was checked using systemd and the Wazuh control utility. Network listeners were also inspected to identify the Manager API, agent communication, enrollment, Dashboard, Indexer, and cluster-related ports.

Port `1516` was observed listening in one captured network-state inspection. This observation is not treated as proof of sustained cluster health or multi-node communication.

### 4. Log Analysis

Manager logs contained repeated `indexer-connector` initialization warnings and agent synchronization failures. Additional entries reported Indexer connection errors.

These findings were recorded separately from the cluster assessment because Indexer connectivity and cluster synchronization are different operational functions.

### 5. Configuration Change and Recovery

A single-node master configuration was attempted by changing the cluster-enabled setting. The modified XML passed `xmllint` validation, but the Wazuh Manager failed to restart.

The previous configuration was restored from:

`/var/ossec/etc/ossec.conf.pre-cluster`

The failed configuration was preserved separately as:

`/var/ossec/etc/ossec.conf.cluster-failed`

After restoration, the Manager service returned to `active`. The Wazuh control utility reported `wazuh-clusterd not running`, which was expected with the restored standalone configuration.

## Key Findings

- The deployment contained one Wazuh VM; no second Manager node was available.
- The original configuration had clustering disabled.
- A captured cluster-control result showed a single master node but no completed synchronization for connected nodes.
- XML syntax validation did not guarantee successful Wazuh startup.
- The cluster-enablement attempt resulted in a Manager restart failure.
- Restoring the previous configuration recovered the Manager service.
- The final confirmed state after rollback was a running standalone Manager.
- Multi-node communication and synchronization could not be validated.
- IndexerConnector warnings and connection errors were also present and require separate investigation.

## Final Assessment

The Wazuh Manager was successfully recovered after the unsuccessful cluster-enablement attempt. Its standalone operational state was restored, and the core Manager processes were running.

The exercise provided evidence of local Manager health and cluster configuration behavior, but it did not establish multi-node cluster health or node consistency.

**Assessment limitation:** A single-Manager environment cannot demonstrate worker connectivity, master-worker synchronization, or cross-node consistency. These checks require an additional Wazuh Manager node.

## Scope and Safety

This lab involved configuration inspection, service checks, log review, a controlled cluster-enablement attempt, and configuration recovery.

It did not include:
- Adding a second Wazuh Manager.
- Establishing a production or multi-node cluster.
- Testing cross-node synchronization.
- Performing a cluster failover.
- Changing Indexer configuration or repairing Indexer connectivity.
- Claiming successful multi-node consistency validation.

Sensitive cluster keys and credentials must not be included in screenshots, investigation reports, or repository files.
