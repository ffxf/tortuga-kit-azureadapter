## Microsoft Azure resource adapter kit

### Overview

As with other cloud providers, Tortuga provides [Microsoft Azure][azure]
support through the use of a resource adapter.

The Tortuga/Azure integration is currently limited to creation of Virtual
Machines (VMs). It will not manage, nor interfere with, other Azure resources,
such as storage accounts, virtual networks/subnets/network security groups,
etc.  These resources must be created prior to using the Tortuga/Azure
integration.

**Note:** the Azure resource adapter requires RHEL/CentOS 7 and will not work
on older OS versions. This is a limitation of the Microsoft Azure Python SDK
which does not support Python 2.6 (the distribution version of Python included
with RHEL/CentOS 6). Tortuga is, however, able to deploy RHEL/CentOS 6 compute
nodes.

Please refer to official [Microsoft Azure
documentation](https://docs.microsoft.com/en-us/azure/) for further explanation
of Azure terms referenced within this document.

### Setting up Azure for Tortuga

Before using Tortuga with the [Microsoft Azure][azure] resource adapter, it is
necessary to create resources within the Azure environment.

1. Obtain Azure credentials

    This requires setting up an application under [Azure Active Directory](https://www.microsoft.com/en-ca/cloud-platform/azure-active-directory).

    A helpful tutorial can be found [here](https://auth0.com/docs/connections/enterprise/azure-active-directory) and from Microsoft ["Integrating applications with Azure Active Directory"](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-integrating-applications)

    The Azure resource adapter requires the following:

    * Client ID
    * Subscription ID
    * Tenant ID
    * Secret

    These values can be obtained from the Azure portal as part of the app
    registration in Azure Active Directory.

1. Install Azure CLI 2.0

    For ease of use, it is *strongly recommended* to install the Azure CLI
    into the Tortuga (Python 2.7) environment as follows:

        /opt/tortuga/bin/pip install azure-cli

    The Azure CLI can also be installed on other host(s) running Linux,
    Windows, or MacOS.

    Official Microsoft documentation is available from [https://docs.microsoft.com/en-us/cli/azure/install-azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

1. Create resource group, as necessary.

    Tortuga can use an existing Azure resource group or a new resource group can be created.

    In this example, the resource group is named *"uc-cluster"* and the location is *"canadacentral"*.

        az group create --name uc-cluster --location canadacentral

    **Hint:** use `az account list-locations --query "[].name"` to query available locations.

1. Create virtual network *vnet1* in resource group *uc-cluster*, as necessary

        az network vnet create --resource-group uc-cluster \
            --location canadacentral --name vnet1

1. Create network security group *tortugansg*, as necessary

        az network nsg create --resource-group uc-cluster \
            --location canadacentral --name tortugansg

    Allow incoming ssh (22/tcp) connections

        az network nsg rule create --resource-group uc-cluster \
            --nsg-name tortugansg --name ssh --priority 100 \
            --destination-address-prefix "*" \
            --destination-port-range 22 --access Allow \
            --protocol Tcp --description "Allow incoming ssh"

    If setting up a hybrid Tortuga environment and using the built-in OpenVPN
    point-to-point VPN support, also allow incoming OpenVPN (`1194/udp`)
    connections as follows:

        az network nsg rule create --resource-group uc-cluster \
            --nsg-name tortugansg --name openvpn --priority 100 \
            --destination-address-prefix "*" \
            --destination-port-range 1194 --access Allow \
            --protocol Udp --description "Allow incoming OpenVPN"

1. Create subnet *subnet1* in virtual network *vnet1*

        az network vnet subnet create \
            --resource-group uc-cluster --vnet-name vnet1 \
            --name subnet1 \
            --address-prefix 10.0.0.0/24 \
            --network-security-group tortugansg

1. Create storage account *tortugastorage* in resource group *uc-cluster*

        az storage account create --resource-group uc-cluster \
            --location canadacentral --sku Premium_LRS \
            --kind Storage --name tortugastorage

### Installing the Azure resource adapter

The Azure resource adapter is distributed via the Azure resource adapter kit
included with the Tortuga distribution.

**Note:** it is not necessary to extract the contents of the `tar.bz2` file in
order to install the kit.

Install the Azure resource adapter kit by running the following command as
`root` on a Tortuga installer host:

    install-kit --i-accept-the-eula kit-azureadapter-6.3.1-0.tar.bz2

The Azure resource adapter is enabled as follows, again run as `root`:

    enable-component -p azureadapter-6.3.1-0 management-6.3
    /opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --verbose

The Azure resource adapter kit is now installed and ready to be configured.

### Azure resource adapter configuration

As with all other resource adapters in Tortuga, the resource adapter must
be configured with cloud provider specific settings. This minimally
includes the Azure access credentials, resource group, storage account,
location, VM size, virtual network, subnet, security group, etc.

Use the `adapter-mgmt` tool to create/update the resource adapter
configuration profile.

1. Create the `default` resource adapter configuration profile for Azure:

    This example configures the Azure resource adapter to use Ubuntu 16.04
    (Xenial) compute nodes in a hybrid environment (on-premise Tortuga
    installer).

        adapter-mgmt create --resource-adapter azure --profile default \
            -s subscription_id=<Azure subscription id> \
            -s client_id=<client id> \
            -s tenant_id=<tenant id> \
            -s secret="<secret>" \
            -s resource_group=<Azure resource group> \
            -s storage_account=<Azure storage account> \
            -s location=<location> \
            -s size=<azure VM size> \
            -s default_login=ubuntu \
            -s security_group=<Azure security group name> \
            -s virtual_network_name=<Azure virtual network name> \
            -s subnet_name=<Azure subnet name> \
            -s image_urn=Canonical:UbuntuServer:16.04.0-LTS:latest \
            -s cloud_init_script_template=ubuntu_bootstrap.py.tmpl

    **Note:** the CentOS images provided by OpenLogic do not enable
    `cloud-init` or the Microsoft Azure Linux Guest Agent (aka *waagent*).
    This prevents them for being used as Tortuga-managed compute nodes as
    there is no mechanism to automatically run a boot script.

    Resource adapter configuration profiles can be updated using
    `adapter-mgmt update`.

    Use `adapter-mgmt show -r azure -p default` to display current settings:

        [root@tortuga ~]# adapter-mgmt show -r azure -p default
        Resource adapter: azure
        Profile: default
        Configuration:
          - client_id = <REDACTED>
          - default_login = ubuntu
          - image_urn = Canonical:UbuntuServer:16.04.0-LTS:latest
          - location = canadacentral
          - resource_group = tortuga
          - secret = <REDACTED>
          - security_group = tortugansg
          - size = Basic_A0
          - storage_account = tortugastorage1
          - subnet_name = default
          - subscription_id = <REDACTED>
          - tenant_id = <REDACTED>
          - cloud_init_script_template = ubuntu_bootstrap.py.tmpl
          - virtual_network_name = vnet1

1. Copy example Ubuntu bootstrap script into place

    This is a sample, end-user modifiable bootstrap script for Tortuga-managed compute nodes.

        cp $TORTUGA_ROOT/kits/kit-azureadapter-6.3.1-0/ubuntu_bootstrap.py.tmpl \
            $TORTUGA_ROOT/config/

    Compute nodes will not converge (join the Tortuga-managed cluster) if this script is not copied into place.

### Creating Azure hardware profile

Create a hardware profile named `azure` for all Azure nodes:

    create-hardware-profile --name azure
    update-hardware-profile --name azure --resource-adapter azure \
        --location remote

### Azure resource tagging

User-defined tags are automatically added to all instances. Tags in [Microsoft Azure][azure] can be used to classify or group resources. For example, to clearly identify all resources within in the same cluster.

They should be specified as key-value pairs in the format `key:value`. Multiple tags must be separated by spaces.

For keys and/or values containing spaces, enclose the spaces in double-quotes.

Example:

    adapter-mgmt update --resource-adapter azure \
        --profile default \
        --setting "tags=owner:admin"

Tag name/values containing spaces:

    adapter-mgmt update --resource-adapter azure \
        --profile default \
        --setting tags="key:value \"this is the tag name:this is the tag value\""

\newpage

### Azure resource adapter configuration reference

#### Required settings

The following Azure resource adapter configuration settings are **mandatory**.

* `subscription_id`

    Azure subscription ID; obtainable from `azure` CLI or Management Portal

* `client_id`

    Azure client ID; obtainable from `azure` CLI or Management Portal

* `tenant_id`

    Azure tenant ID; obtainable from `azure` CLI or Management Portal

* `secret`

    Azure client secret; obtainable from `azure` CLI or Management Portal

* `resource_group`

    Azure resource group where Tortuga will create virtual machines

* `storage_account`

    Azure storage account where virtual disks for Tortuga-managed nodes will be created

* `location`

    Azure region in which to create virtual machines

* `size`

    "Size" of virtual machine instances

    reference: [Sizes for Linux virtual machines in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-linux-sizes)

* `default_login`

    Default user login on compute nodes. A login is created by default on
    Tortuga-managed compute nodes for the specified user.

    See description of `ssh_key_value` setting below for details on further
    information on connecting to Azure VMs deployed by Tortuga using SSH
    key-based authentication.

* `virtual_network_name`

    Name of virtual network to associate with virtual machines

* `subnet_name`

    Name of subnet to be used within configured virtual network

* `image_urn`

    URN of desired operating system VM image

    **Note:** `image` and `image_urn` are mutually exclusive settings. Use `image_urn` for specifying a VM image from the Azure repository.

* `image`

    Name of VM image

    **Note:** `image` and `image_urn` are mutually exclusive settings. Use `image` for configuring new VMs to use a VM image available in the resource group.

* `cloud_init_script_template`

    Use this setting to specify the filename/path of the [`cloud-init`][cloud_init] script template. If the path is not fully-qualified (does **not** start with a leading forward slash), it is assumed the script path is `$TORTUGA_ROOT/config`.

    **Note:** this setting is only applies to VM images that are
    [`cloud-init`][cloud-init] enabled (ie. the official Ubuntu VM image
    provided on Azure). The OpenLogic-provided CentOS VM images on Azure are
    **not** `cloud-init` enabled.

* `user_data_script_template`

    File name of bootstrap script template to be used on compute nodes. If the path is not fully-qualified (ie. does **not** start with a leading forward slash), it is assumed the script path is `$TORTUGA_ROOT/config`

    This is the script run by [WALinuxAgent][] when VMs are initially booted.

    **Note:** the official OpenLogic-provided CentOS VM images on Azure are [WALinuxAgent][] enabled, however do not enable the ability to launch a bootstrap script. A custom or alternative VM image with this functionality enabled must be used. Consult the official [WALinuxAgent][] reference for more information.

#### Optional settings

* `allocate_public_ip`

    Default: "true"

    When disabled (value "false"), VMs created by Tortuga will not have a public IP address.

* `storage_account_type`

    Use specified storage account type when using an VM image.

    Current default is to use "Standard_LRS". Options are "Standard_LRS" or "Premium_LRS".

    "Premium_LRS" provides support for higher performance SSD backed OS disks.

* `tags`

    Space-separated "key=value" pairs.

* `vpn`

    Default is `false` (disabled)

    Enable point-to-point VPN connections between Tortuga installer using OpenVPN.

    VPN **should not** be enabled when the Tortuga installer is hosted on Azure.

    Do not enable `vpn` setting if using an externally managed site-to-site VPN solution.

* `override_dns_domain`

    Valid values: "true" or "false"

    If `override_dns_domain` is set to `true`, Azure-based compute nodes
    managed by Tortuga will be assigned a DNS host name generated using the
    private DNS zone. The can be set using the Tortuga CLI `set-private-dns-zone`
    and defaults to `private`.

    If `override_dns_domain` is not explictly set to `false` and `dns_domain` is
    defined, DNS domain override will be automatically enabled.

    If `override_dns_domain` is set to `false`, Tortuga does not set the
    compute nodes' host name or alter `/etc/resolv.conf` and depends on the Azure
    defaults.

* `dns_domain`

    This setting overrides the Tortuga private DNS zone setting, which can be set
    using `set-private-dns-zone`.

    If `dns_domain` is set, Azure compute node host names are generated using
    the hardware profile host name format with this DNS domain suffix.

    Example, if `dns_domain` is defined as `cloud.mydomain.com` and the hardware
    profile name format is `compute-#NN`, a possible host name is:

        compute-01.cloud.mydomain.com

    If `override_dns_domain` is enabled and `dns_domain` is *not* set, the host name
    is derived from the value of the Tortuga private DNS zone:

        compute-01.private

* `dns_search`

    Set searchl list for compute node host name lookup. Default is the private
    DNS domain suffix if 'override_dns_domain' is enabled, otherwise DNS domain
    suffix of Tortuga installer.

* `dns_nameservers`

    Space-separated list of IP addresses to be set in `/etc/resolv.conf`

    This setting overrides "DNS Servers" setting of the Azure Virtual Network.

    If `dns_nameservers` is not defined, the default is Tortuga DNS server IP.

* `ssh_key_value`

    Specifies the SSH public key or public key file path. If the value of this
    setting starts with a forward slash (/), it is assumed to be a file path.
    The SSH public key will be read from this file.

    If `ssh_key_value` is undefined, the SSH public key of the `root` user (normally
    `/root/.ssh/id_rsa.pub`) is used.

    This setting works in conjunction with the `default_login` setting to allow
    secure SSH key-based authentication between the Tortuga installer and Azure
    compute nodes deployed by Tortuga.

* `vcpus`

    Default behaviour is to use the virtual CPUs count obtained from Azure. If `vcpus`
    is defined, it overrides the default value.

    When the Azure resource adapter is used in conjunction with Univa Grid Engine, this
    value will be used to automatically configure Grid Engine the exechost slots.

\newpage

### Azure VM image requirements

[Microsoft Azure][azure] virtual machines use Microsoft Azure Linux Guest Agent (aka `waagent`) as a mechanism to pass user scripts and metadata into Azure virtual machine instances at launch time.

Extraction and execution of scripts contained within metadata is disabled in the `waagent` configuration by default.

The Ubuntu VM images provided enable both `waagent` as well as `cloud-init`.

Any end-user provided VM images are recommended to include support for both
`waagent` as well as `cloud-init`. Tortuga requires `cloud-init` to properly
bootstrap Azure VMs and incorporate them into the Tortuga-managed cluster.

\newpage

### Azure security group requirements

For hybrid Tortuga installations, it is recommended the network security group minimally allows `ssh` (22/tcp) connections. If using the built-in point-to-point OpenVPN, it is also required to allow port 1193/udp.

Azure network security groups are created with open access for outbound network traffic and unrestricted connectivity to virtual machines within the same network/subnet.

\newpage

### DNS and Azure

The default built-in Azure DNS server implements *only* forward DNS. Univa Grid
Engine requires both forward and reverse (IP to host name) DNS resolution. As a
result, it is necessary to enable the built-in Tortuga DNS server as follows:

    enable-component -p dns
    genconfig dns
    /opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --verbose

Refer to the section in this manual on configuring and/or customizing the
built-in Tortuga DNS server for more information.

**Note:** if the Tortuga DNS server is enabled *after* the UGE qmaster has
been started on the Tortuga installer, it will be necessary to restart
(stop/start) the UGE qmaster:

On RHEL/CentOS 6:

    service sgemaster.tortuga stop
    service sgemaster.tortuga.stop

On RHEL/CentOS 7:

    systemctl stop sgemaster.tortuga
    systemctl start sgemaster.tortuga

If using an external/custom DNS server, ensure it provides forward and reverse
DNS resolution for Tortuga managed nodes.

#### Override default Azure DNS settings

Override the default Azure DNS settings by setting `override_dns_domain` in the
Azure resource adapter configuration:

    adapter-mgmt update --resource-adapter azure --profile default \
        -s override_dns_domain=true \
        -s dns_domain cloud.univa.com

If `override_dns_domain` is enabled (set to `true`) and `dns_domain` is **not**
set, the global Tortuga private DNS domain will be used. This can be
queried/set using the Tortuga CLI`set-private-dns-zone`.

Refer to documentation on Tortuga DNS for further details.

Add `dns_search` and `dns_nameservers` settings here as appropriate. For
example, if the corporate DNS server enables DNS (sub)domain delegation for
Tortuga-managed nodes, it may be desirable to set `dns_nameservers` to
**only** include the IP address(es) of the corporate DNS server(s). This would
configure Tortuga-managed Azure compute nodes to use the upstream corporate
DNS server, which would then delegate DNS lookups to the Tortuga DNS server.

**Note:** it is possible to set the default DNS server IP address (but not the
DNS domain) in the Azure Virtual Network settings. This DNS server setting is
applied **unless** `override_dns_domain` is enabled.

\newpage

### Azure resource adapter usage

#### Supported node operations

The [Microsoft Azure][azure] resource adapter supports the following Tortuga node management commands:

- `activate-node`
- `add-nodes`
- `delete-node`
- `idle-node`
- `reboot-node`
- `transfer-node`
- `shutdown-node`
- `startup-node`

The Azure resource adapter *does not* support the following node operation commands as they do not make sense within the context of cloud-based compute nodes:

- `checkpoint-node`
- `migrate-node`

#### Networking considerations

The Azure resource Adapter automatically configures an [OpenVPN][] virtual private network when the Tortuga installer is installed on-premise. The VPN is not automatically configured if the Tortuga installer itself is running on Azure.

The VPN requires a network address space in which to create compute nodes. For a hybrid environment where the installer is on-premise and the compute nodes are entirely cloud-based, this network can be attached to a interface defined by Ethernet alias or VLAN(ie. `eth0:0`). It does not need to be an actual physical network connected to a physical Ethernet interface.

To enable a *true* hybrid environment with on-premise compute nodes (physical and/or virtual) and cloud-based compute nodes, the network used for the VPN needs to be the same as the network where the local compute nodes are connected or network routing must be configured appropriately.

#### Adding Azure nodes

Assuming hardware and software profiles have been created as described above, adding nodes in the Azure environment is done using the `add-nodes` command.

The following example will create 6 nodes on Azure using the software profile `execd` and hardware profile `execd-azure`.

    add-nodes --count 6 \
        --software-profile execd \
        --hardware-profile execd-azure

It is assumed the software profile `execd` is mapped to hardware profile `execd-azure` and the hardware profile `execd-azure` is properly configured as per the above.

If using a resource adapter configuration profile, use the `--resource-adapter-configuration` (or the `-A` shortcut) argument:

    add-nodes --count 6 \
        --software-profile execd \
        --hardware-profile execd-azure \
        --resource-adapter-configuration otherzone

where *otherzone* is the name of an existing Azure resource adapter configuration profile.

#### Extra `add-nodes` arguments for Azure

It is possible to specify the public SSH key per invocation of `add-nodes`. This works
similarly to the Azure CLI `--ssh-key-value` argument.

For example, to add nodes using the SSH public key found in the file `/root/.ssh/my_pub_key`:

    add-nodes --count 6 \
        --software-profile execd \
        --hardware-profile execd-azure \
        --extra-arg="ssh-key-value=/root/.ssh/my_pub_key"

Similar to the resource adapter configuration setting, it is also possible to specify
the key here as well:

    add-nodes --count 6 \
        --software-profile execd \
        --hardware-profile execd-azure \
        --extra-arg="ssh-key-value=\"ssh-rsa ... mykey\""

where the ellipsis (...) is the actual key value.

The `ssh_key_value` setting can also be applied to the resource adapter configuration.

\newpage

### Best Practices

* Create a unique Azure resource group and storage accounts for Tortuga to prevent "cross-polination" of Tortuga-managed resources with other resources within the same [Microsoft Azure][azure] account.

\newpage

### Advanced Topics

#### Hosting Tortuga installer on Microsoft Azure

Tortuga can be hosted entirely on [Microsoft Azure][azure]. This entails running a dedicated Azure VM instance for the Tortuga installer, which will then be able to provision compute nodes.

`Standard_DS2_v2` is the recommended minimum Azure VM size to be used for the Tortuga installer. Azure VM sizes may vary across Azure datacenter locations.

\newpage

### Troubleshooting

There are currently no Azure-specific debug options within Tortuga. Due to the newness of the Azure resource adapter, debug logging in Tortuga is currently verbose by default.

Refer to the Tortuga log file (`/var/log/tortuga`) for further information on failed operations.

\newpage

[azure]:              https://azure.microsoft.com                   "Microsoft Azure"
[WALinuxAgent]:       https://github.com/Azure/WALinuxAgent         "WALinuxAgent"
