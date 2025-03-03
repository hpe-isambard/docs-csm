# BOS Workflows

The following workflows present a high-level overview of common Boot Orchestration Service \(BOS\) operations.
These workflows depict how services interact with each other when booting, configuring, or shutting down nodes.
They also help provide a quicker and deeper understanding of how the system functions.

* [Terminology](#terminology)
* [Workflows]
    * [Boot nodes](#boot-nodes)
    * [Reboot nodes](#reboot-nodes)
    * [Power off nodes](#power-off-nodes)

## Terminology

The following are mentioned in the workflows:

* Boot Orchestration Service \(BOS\) is responsible for booting, configuring, and shutting down collections of nodes.
  The Boot Orchestration Service has the following components:
    * A BOS session template is a collection of one or more boot sets. A boot set defines a collection of nodes and the information about the boot artifacts and parameters.
      Session templates also include information on which [Configuration Framework Service (CFS)](../../glossary.md#configuration-framework-service-cfs) configuration should
      be applied.
    * BOS sessions provide a way to apply a template across a group of nodes and monitor the progress of those nodes as they move toward their desired state.
    * BOS operators interact with other services to perform actions on nodes, moving them toward their desired state.
* [Power Control Service (PCS)](../../glossary.md#power-control-service-pcs) provides system-level power control
  for nodes in the system. PCS interfaces directly with the Redfish APIs to the controller infrastructure to effect power and environmental changes on the system.
* [Hardware State Manager (HSM)](../../glossary.md#hardware-state-manager-hsm) tracks the state of each node and its group and role associations.
* [Boot Script Service (BSS)](../../glossary.md#boot-script-service-bss) stores per-node information about the iPXE boot script.
  When booting or rebooting, nodes consult BSS for boot artifacts \(kernel, `initrd`, image root\) and boot parameters.
* [Simple Storage Service (S3)](../../glossary.md#simple-storage-service-s3) is an artifact repository that stores boot artifacts.
* CFS configures nodes using the configuration framework. Launches and aggregates the status from one or more Ansible instances against nodes
  \(node personalization\) or images \(image customization\).

## Workflows

The following workflows are included in this section:

* [Boot nodes](#boot-nodes)
* [Reboot nodes](#reboot-nodes)
* [Power off nodes](#power-off-nodes)

### Boot nodes

**Use case:** Administrator powers on and configures select compute nodes.

**BOS v2 boot flow diagram:** This labels on the diagram correspond to the workflow steps listed below. Some steps are omitted from the diagram for readability.

![Boot Nodes](../../img/operations/boot_orchestration/bos_v2_boot.png)

**Workflow overview:** The following sequence of steps occurs during this workflow.

1. **Administrator creates a configuration**

    (`ncn-mw#`) Add a configuration to CFS. For more information on creating CFS configurations, see [CFS Configurations](../configuration_management/CFS_Configurations.md).

    ```bash
    cray cfs v3 configurations update sample-config --file configuration.json --format json
    ```

    Example output:

    ```json
    {
        "last_updated": "2020-09-22T19:56:32Z",
        "layers": [
            {
                "clone_url": "https://api-gw-service-nmn.local/vcs/cray/configmanagement.git",
                "commit": "01b8083dd89c394675f3a6955914f344b90581e2",
                "playbook": "site.yaml"
            }
        ],
        "name": "sample-config"
    }
    ```

1. **Administrator creates a BOS session template**

    A session template is a collection of data specifying a group of nodes, as well as the boot artifacts and configuration that should be applied to them.
    A session template can be created from a JSON structure. It returns a session template ID if successful.

    See [Manage a session template](Manage_a_Session_Template.md) for more information.

1. **Administrator creates a session**

    Create a session to perform the operation specified in the operation request parameter on the boot set defined in the session template. For this use case,
    the administrator creates a session with operation as `boot` and specifies the session template ID.

    (`ncn-mw#`)

    ```bash
    cray bos v2 sessions create --template-name SESSIONTEMPLATE_NAME --operation boot
    ```

1. **Session setup operator**

    The creation of a session causes the session-setup operator to set a desired state on all components listed in the session template.
    This includes pulling files from S3 to determine boot artifacts like kernel, `initrd`, and root file system. The session setup operator also enables the relevant
    components at this time.

1. **Status operator (powering-on)**

    The status operator will detect the enabled components and assign them a phase. This involves checking the current state of the node, including communicating with HSM
    to determine the current power status of the node.

    In this example of booting nodes, the first phase is `powering-on`. If queried at this point, the nodes will have a status of `power-on-pending`.
    For more on component phase and status, see [Component Status](Component_Status.md)

1. **Power-on operator**

    The power-on operator will detect nodes with a `power-on-pending` status. If root file system provider is the Scalable Boot Provisioning Service (`SBPS`), then the power-on operator
    will notify SBPS that the root file system needs to be projected. It will do this by tagging the image with `sbps-project: true` using the Image Management Service (IMS).
     Then, the power-on operator sets the desired boot artifacts in BSS.
    If configuration is enabled for the node, the power-on operator will also call CFS to set the desired configuration and disable the node with CFS.
    The node must be disabled within CFS so that CFS does not try to configure node until it has booted.
    The power-on operator then calls PCS to power-on the node.
    Lastly, the power-on operator will update the state of the node in BOS, including setting the last action. If queried at this point,
    the nodes will have a status of `power-on-called`.

1. **PCS boots nodes**

    PCS interfaces directly with the Redfish APIs and powers on the selected nodes.

1. **BSS interacts with the nodes**

    BSS generates iPXE boot scripts based on the image content and boot parameters that have been assigned to a node. Nodes download the iPXE boot script from BSS.

1. **Nodes request boot artifacts from S3**

    Nodes download the boot artifacts. The nodes boot using the boot artifacts pulled from S3.

1. **Status operator (configuring)**

    The status operator monitors a node's power state until HSM reports that the power state is on.
    When the power state for a node is on, the status operator will either set the phase to `configuring` if CFS configuration is required or it will clear the current phase
    if the node is in its final state.

1. **CFS applies configuration**

    If needed, CFS runs Ansible on the nodes and applies post-boot configuration \(also called node personalization\).

1. **Status operator (complete)**

    The status operator will continue monitoring the states for each node until CFS reports that configuration is complete.
    The status operator will clear the current phase now that the node is in its final state. The status operator will also disable components at this point.

1. **Session completion operator**

    When all nodes belonging to a session have been disabled, the session is marked complete, and its final status is saved to the database.

### Reboot nodes

**Use case:** Administrator reboots and configures select compute nodes.

**BOS v2 reboot flow diagram:** This labels on the diagram correspond to the workflow steps listed below. Some steps are omitted from the diagram for readability.

![Boot Nodes](../../img/operations/boot_orchestration/bos_v2_reboot.png)

**Workflow overview:** The following sequence of steps occurs during this workflow.

1. **Administrator creates a configuration**

    Add a configuration to CFS. For more information on creating CFS configurations, see [CFS Configurations](../configuration_management/CFS_Configurations.md).

    (`ncn-mw#`)

    ```bash
    cray cfs v3 configurations update sample-config --file configuration.json --format json
    ```

    Example output:

    ```json
    {
        "last_updated": "2020-09-22T19:56:32Z",
        "layers": [
            {
                "clone_url": "https://api-gw-service-nmn.local/vcs/cray/configmanagement.git",
                "commit": "01b8083dd89c394675f3a6955914f344b90581e2",
                "playbook": "site.yaml"
            }
        ],
        "name": "sample-config"
    }
    ```

1. **Administrator creates a BOS session template**

    A session template is a collection of data specifying a group of nodes, as well as the boot artifacts and configuration that should be applied to them.
    A session template can be created from a JSON structure. It returns a session template ID if successful.

    See [Manage a session template](Manage_a_Session_Template.md) for more information.

1. **Administrator creates a session**

    Create a session to perform the operation specified in the operation request parameter on the boot set defined in the session template. For this use case,
    the administrator creates a session with operation as `reboot` and specifies the session template ID.

    (`ncn-mw#`)

    ```bash
    cray bos v2 sessions create --template-name SESSIONTEMPLATE_NAME --operation reboot
    ```

1. **Session setup operator**

    The creation of a session causes the session-setup operator to set a desired state on all components listed in the session template.
    This includes pulling files from S3 to determine boot artifacts like kernel, `initrd`, and root file system. The session setup operator also enables the relevant
    components at this time.

1. **Status operator (powering-off)**

    The status operator will detect the enabled components and assign them a phase. This involves checking the current state of the node, including communicating with
    HSM to determine the current power status of the node.

    In this example of rebooting nodes, the first phase is `powering-off`. If queried at this point, the nodes will have a status of `power-off-pending`.
    For more on component phase and status, see [Component Status](Component_Status.md)

1. **Graceful-power-off operator**

    The power-off operator will detect nodes with a `power-off-pending` status, calls PCS to power-off the node.
    Then, the power-off operator will update the state of the node in BOS, including setting the last action. If queried at this point, the nodes will have a status of
    `power-off-gracefully-called`.

1. **Forceful-power-off operator**

    If powering-off is taking too long, the forceful-power-off will take over. It also calls PCS to power-off the node, but with the addition of the forceful flag.
    Then, the power-off operator will update the state of the node in BOS, including setting the last action. If queried at this point, the nodes will have a status of
    `power-off-forcefully-called`.

1. **PCS powers off nodes**

    PCS interfaces directly with the Redfish APIs and powers off the selected nodes.

1. **Status operator (powering-on)**

    The status operator monitors a node's power state until HSM reports that the power state is off.
    When the power state for a node is off, the status operator will set the phase to `powering-on`. If queried at this point, the nodes will have a status of
    `power-on-pending`.

1. **Power-on operator**

    The power-on operator will detect nodes with a `power-on-pending` status. If root file system provider is the Scalable Boot Provisioning Service (`SBPS`), then the power-on operator
    will notify SBPS that the root file system needs to be projected. It will do this by tagging the image with `sbps-project: true` using the Image Management Service (IMS).
     Then, the power-on operator sets the desired boot artifacts in BSS.
    If configuration is enabled for the node, the power-on operator will also call CFS to set the desired configuration and disable the node with CFS.
    The node must be disabled within CFS so that CFS does not try to configure node until it has booted.
    The power-on operator then calls PCS to power-on the node.
    Lastly, the power-on operator will update the state of the node in BOS, including setting the last action. If queried at this point, the nodes will have a status of
    `power-on-called`.

1. **PCS boots nodes**

    PCS interfaces directly with the Redfish APIs and powers on the selected nodes.

1. **BSS interacts with the nodes**

    BSS generates iPXE boot scripts based on the image content and boot parameters that have been assigned to a node. Nodes download the iPXE boot script from BSS.

1. **Nodes request boot artifacts from S3**

    Nodes download the boot artifacts. The nodes boot using the boot artifacts pulled from S3.

1. **Status operator (configuring)**

    The status operator monitors a node's power state until HSM reports that the power state is on.
    When the power state for a node is on, the status operator will either set the phase to `configuring` if CFS configuration is required or it will clear the current
    phase if the node is in its final state.

1. **CFS applies configuration**

    If needed, CFS runs Ansible on the nodes and applies post-boot configuration \(also called node personalization\).

1. **Status operator (complete)**

    The status operator will continue monitoring the states for each node until CFS reports that configuration is complete.
    The status operator will clear the current phase now that the node is in its final state. The status operator will also disable components at this point.

1. **Session completion operator**

    When all nodes belonging to a session have been disabled, the session is marked complete, and its final status is saved to the database.

### Power off nodes

**Use case:** Administrator powers off selected compute nodes.

**BOS v2 Shutdown flow diagram:** This labels on the diagram correspond to the workflow steps listed below. Some steps are omitted from the diagram for readability.

![Boot Nodes](../../img/operations/boot_orchestration/bos_v2_shutdown.png)

**Workflow overview:** The following sequence of steps occurs during this workflow.

1. **Administrator creates a BOS session template**

    A session template is a collection of data specifying a group of nodes, as well as the boot artifacts and configuration that should be applied to them. A session
    template can be created from a JSON structure. It returns a session template ID if successful.

    See [Manage a session template](Manage_a_Session_Template.md) for more information.

1. **Administrator creates a session**

    Create a session to perform the operation specified in the operation request parameter on the boot set defined in the session template. For this use case,
    the administrator creates a session with operation as `shutdown` and specifies the session template ID.

    (`ncn-mw#`)

    ```bash
    cray bos v2 sessions create --template-name SESSIONTEMPLATE_NAME --operation shutdown
    ```

1. **Session setup operator**

    The creation of a session causes the session-setup operator to set a desired state on all components listed in the session template.
    For a power-off, this means clearing the desired state for each component.
    The session setup operator also enables the relevant components at this time.

1. **Status operator (powering-off)**

    The status operator will detect the enabled components and assign them a phase. This involves checking the current state of the node, including communicating
    with HSM to determine the current power status of the node.

    In this example of booting nodes, the first phase is `powering-off`. If queried at this point, the nodes will have a status of `power-off-pending`.
    For more on component phase and status, see [Component Status](Component_Status.md)

1. **Graceful-power-off operator**

    The power-off operator will detect nodes with a `power-off-pending` status, calls PCS to power-off the node.
    Then, the power-off operator will update the state of the node in BOS, including setting the last action. If queried at this point, the nodes will have a status of
    `power-off-gracefully-called`.

1. **Forceful-power-off operator**

    If powering-off is taking too long, the forceful-power-off will take over. It also calls PCS to power-off the node, but with the addition of the forceful flag.
    Then, the power-off operator will update the state of the node in BOS, including setting the last action. If queried at this point, the nodes will have a status of
    `power-off-forcefully-called`.

1. **PCS powers off nodes**

    PCS interfaces directly with the Redfish APIs and powers off the selected nodes.

1. **Status operator (complete)**

    The status operator will continue monitoring the states for each node until CFS reports that configuration is complete.
    The status operator will clear the current phase now that the node is in its final state. The status operator will also disable components at this point.

1. **Session completion operator**

    When all nodes belonging to a session have been disabled, the session is marked complete, and its final status is saved to the database.
