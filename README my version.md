# Wazuh-Cluster-Health-Node-Consistency
## Overview
Wazuh cluster health monitoring is the process of checking whether the Wazuh Manager cluster is functioning correctly and whether its nodes are communicating and maintaining consistent operational information.

A Wazuh cluster uses a **master node** and one or more **worker nodes**:

- **Master node:** Coordinates cluster operations and manages shared cluster information.
- **Worker nodes:** Handle agent connections and process events while communicating with the master.
- **Cluster communication:** Allows nodes to exchange information required for coordinated operation.
- **Node consistency:** Refers to whether nodes have the expected configuration, cluster status, and operational state.

A cluster can have running services but still experience problems such as disconnected workers, delayed synchronization, inconsistent configuration, or communication failures. Therefore, checking service status alone is not sufficient to establish cluster health.

### What Should Be Checked?

| Area | What It Tells Us |
|---|---|
| Cluster status | Whether the cluster is enabled and reporting its state |
| Node membership | Which nodes are registered and visible to the cluster |
| Master/worker roles | Whether nodes have the expected roles |
| Node connectivity | Whether workers can communicate with the master |
| Synchronization | Whether cluster information is being shared as expected |
| Version consistency | Whether cluster nodes use compatible Wazuh versions |
| Service health | Whether the Manager service is running on each node |
| Agent distribution | Whether agents are connected to the expected nodes |

**Important distinction:** A healthy cluster requires more than active services. Node membership, communication, synchronization, and version compatibility must also be evaluated. A single-node deployment cannot demonstrate master-worker synchronization or multi-node consistency.

This lab examines Wazuh Manager health, cluster configuration, node visibility, and synchronization status in a Rocky Linux-based Wazuh deployment.

The assessment began with a standalone Wazuh Manager configuration in which clustering was disabled. Cluster-control commands, configuration inspection, service checks, network-listener inspection, and log analysis were used to understand the deployment's operational state.

A single-node master configuration was also attempted to explore Wazuh cluster functionality. Although the modified configuration passed XML syntax validation, the Manager failed to restart. The original configuration was subsequently restored, and the Manager service returned to an active state.

The exercise demonstrates why cluster health cannot be established from service status or configuration settings alone. Node membership, communication, synchronization, and the actual deployment topology must also be considered.

## Lab Objectives

The objectives of this lab are to:

- Understand the architecture and operational responsibilities of Wazuh master and worker nodes.
- Identify the current Wazuh deployment topology and determine whether clustering is enabled.
- Examine cluster configuration parameters, including cluster name, node identity, node role, communication port, and configured node addresses.
- Use Wazuh cluster-control utilities to inspect cluster status, node membership, and synchronization information.
- Evaluate the relationship between Manager service availability and actual cluster operational health.
- Inspect network listeners and identify the ports associated with agent communication, enrollment, API access, and cluster communication.
- Review Wazuh Manager logs to identify cluster-related messages, communication issues, and operational warnings.
- Understand the limitations of validating cluster synchronization and node consistency in a single-Manager deployment.
- Examine the impact of enabling cluster mode on the existing standalone Manager configuration.
- Practice configuration backup and recovery following an unsuccessful configuration change.
- Verify Manager service recovery and assess the operational state of its core processes.
- Document verified findings, unresolved issues, and validation limitations without treating a single-node deployment as a fully tested multi-node cluster.

## Lab Scenario

A Wazuh deployment running on a Rocky Linux 9.8 virtual machine is being assessed to understand its cluster configuration, operational health, and node consistency. The environment contains Wazuh Manager, Indexer, and Dashboard version 4.13.1, along with a connected Windows endpoint. However, the deployment currently uses only one Wazuh Manager, and its cluster configuration has clustering disabled.

The investigation begins with an examination of the existing cluster settings, Manager service status, network listeners, and cluster-control output. The initial cluster-control command reports that the cluster is not running. Further configuration inspection confirms that the cluster section identifies the node as a master but has `disabled: yes`.

To explore cluster operation, a single-node master configuration is attempted. The modified configuration passes XML syntax validation, but the Wazuh Manager fails to restart after the change. The previous configuration is restored from its backup, allowing the Manager service to return to an active state. This highlights the importance of configuration backups and the difference between XML validity and successful application-level startup.

During the assessment, additional operational observations require attention:
- A captured cluster listing reports one master node, but no completed synchronization for connected nodes.
- Port `1516` is observed listening during one network inspection, although this does not establish worker connectivity.
- Manager logs contain repeated IndexerConnector initialization warnings, agent synchronization failures, and Indexer connection errors.
- After the rollback, the Manager operates with its original standalone configuration and `wazuh-clusterd` remains stopped.

The investigation must determine what can be verified from the available evidence and distinguish local Manager health from actual cluster health. Since the environment contains only one Wazuh Manager, worker connectivity, master-worker synchronization, and cross-node consistency cannot be meaningfully tested.

**Scenario objective:** Assess the Wazuh deployment's cluster configuration and operational state, examine the consequences of attempting to enable clustering, recover the Manager using the preserved configuration, and document the limitations and outstanding issues. The exercise does not establish a functioning multi-node cluster or attempt to simulate synchronization between separate Manager nodes.

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

