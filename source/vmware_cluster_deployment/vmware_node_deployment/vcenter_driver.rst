.. _vcenterg:
.. _vcenter_setup:

vCenter Driver Setup
====================

The vCenter Driver is the responsible for all the integration of OpenNebula with VMware based infrastructures. All the interaction between OpenNebula and vSphere is channeled through the vCenter API, except for the VNC console connection where Sunstone server (more specifically, the noVNC server) contacts with the ESX hypervisors directly.

This section lays out the actions needed to incorporate VMware resources to an OpenNebula cloud.

Driver Defaults
---------------

vCenter Virtualization Drivers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following sections in the ``/etc/one/oned.conf`` file describe the information and virtualization drivers for vCenter, which are enabled by default:

.. code::

    #-------------------------------------------------------------------------------
    #  vCenter Virtualization Driver Manager Configuration
    #    -r number of retries when monitoring a host
    #    -t number of threads, i.e. number of hosts monitored at the same time
    #    -p more than one action per host in parallel, needs support from hypervisor
    #    -s <shell> to execute commands, bash by default
    #    -w Timeout in seconds to execute external commands (default unlimited)
    #-------------------------------------------------------------------------------
    VM_MAD = [
        NAME              = "vcenter",
        SUNSTONE_NAME     = "VMWare vCenter",
        EXECUTABLE        = "one_vmm_sh",
        ARGUMENTS         = "-p -t 15 -r 0 -s sh vcenter",
        TYPE              = "xml",
        KEEP_SNAPSHOTS    = "yes",
        DS_LIVE_MIGRATION = "yes",
        COLD_NIC_ATTACH   = "yes",
        LIVE_RESIZE       = "yes",
        IMPORTED_VMS_ACTIONS = "terminate, terminate-hard, hold, release, suspend,
            resume, delete, reboot, reboot-hard, resched, unresched, poweroff,
            poweroff-hard, disk-attach, disk-detach, nic-attach, nic-detach,
            snapshot-create, snapshot-delete, migrate, live-migrate"
    ]
    #-------------------------------------------------------------------------------


IMPORTED_VMS_ACTIONS define which operatons are allowed to be executed on imported VMs (Wild VMs).

As a Virtualization driver, the vCenter driver accepts a series of parameters that control its execution. The parameters allowed are:

+----------------+-------------------------------------------------------------------+
| parameter      | description                                                       |
+================+===================================================================+
| -r <num>       | number of retries when executing an action                        |
+----------------+-------------------------------------------------------------------+
| -t <num        | number of threads, i.e. number of actions done at the same time   |
+----------------+-------------------------------------------------------------------+

See the :ref:`Virtual Machine drivers reference <devel-vmm>` for more information about these parameters, and how to customize and extend the OpenNebula virtualization drivers. Also check the specific :ref:`vCenter driver reference <vcenter_driver>`, that details how OpenNebula maps and keeps track of vCenter resources.

Additionally some behavior of the vCenter driver can be configured in the ``/var/lib/one/remotes/etc/vmm/vcenter/vcenterrc``. Check the file to learn what parameters can be changed.

vCenter Datastore Drivers
~~~~~~~~~~~~~~~~~~~~~~~~~

The following section configures the vCenter datastore drivers, used to copy images to and from vCenter image datastores.

.. code::

    DS_MAD_CONF = [
        NAME = "vcenter",
        REQUIRED_ATTRS = "VCENTER_INSTANCE_ID,VCENTER_DS_REF,VCENTER_DC_REF",
        PERSISTENT_ONLY = "NO",
        MARKETPLACE_ACTIONS = "export"
    ]

    DATASTORE_MAD = [
        EXECUTABLE = "one_datastore",
        ARGUMENTS  = "-t 15 -d dummy,fs,lvm,ceph,dev,iscsi_libvirt,vcenter -s shared,ssh,ceph,fs_lvm,qcow2,vcenter"
    ]


The following section configures the vCenter datastore transfer drivers, used to copy images to and from vCenter system datastores.

.. code::

    TM_MAD = [
        EXECUTABLE = "one_tm",
        ARGUMENTS = "-t 15 -d dummy,lvm,shared,fs_lvm,qcow2,ssh,ceph,dev,vcenter,iscsi_libvirt"
    ]

    TM_MAD_CONF = [
        NAME = "vcenter", LN_TARGET = "NONE", CLONE_TARGET = "SYSTEM", SHARED = "YES"
    ]

See the :ref:`Storage Drivers reference <sd>` for more information about these parameters, and how to customize and extend the drivers. The :ref:`vCenter Storage Setup <vmware_storage_setup>` guide gives more details on the vCenter storage model in OpenNebula.

vCenter Networking Drivers
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following section configures the networking drivers, used to import and create networks from vCenter.

.. code::

    VN_MAD_CONF = [
        NAME = "vcenter",
        BRIDGE_TYPE = "vcenter_port_groups"
    ]

See the :ref:`Networking Drivers reference <devel-nm>` for more information about these parameters, and how to customize and extend the drivers. The :ref:`vCenter Networking Setup <vmware_networking_setup>` guide gives more details on the vCenter networking model in OpenNebula.

vCenter Monitoring Drivers
~~~~~~~~~~~~~~~~~~~~~~~~~~

vCenter clusters and Virtual Machines monitoring is performed through ``onemonitord``. Details on its configuration can be found on the :ref:`dedicated guide <mon>`.

See the :ref:`Monitoring Drivers reference <devel-im>` for development information about these drivers and how to customize and extend them.

vCenter Import Tool
--------------------------------------------------------------------------------

vCenter clusters, VM templates, networks, datastores and VMDK files located in vCenter datastores can be easily imported into OpenNebula:

* Using the **onevcenter** tool from the command-line interface

.. prompt:: bash $ auto

    $ onevcenter <command> -o <object type> -h <opennebula host_id> [<options>] [<args]

* Using the Import button in Sunstone.

.. warning:: The image import operation may take a long time. If you use the Sunstone client and receive a "Cannot contact server: is it running and reachable?" the 30 seconds Sunstone timeout may have been reached. In this case either configure Sunstone to live behind Apache/NGINX or use the CLI tool instead.


Following vCenter resources can be easily imported into OpenNebula:

* vCenter clusters (imported as OpenNebula Hosts)
* Datastores
* Networks
* VM Templates
* Wild VMs (VMs launched outside of OpenNebula)
* Images

.. _vcenter_import_clusters:

Importing vCenter Clusters
--------------------------------------------------------------------------------

OpenNebula can import vCenter clusters as OpenNebula host using Sunstone (``Infrastructure-->Hosts``) or through CLI (onevcenter).

This is the only step where vCenter user credentials are required.

Import a vCenter cluster with onevcenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When you select a vCenter cluster to be imported, OpenNebula will create an OpenNebula host that will represent the vCenter cluster. You can instruct OpenNebula in what OpenNebula cluster you want to group the OpenNebula host, if you don't select a previously existing cluster, the default action is to create an OpenNebula cluster for you.

A sample session follows:

.. prompt:: bash $ auto

	$ onevcenter hosts --vcenter <vcenter-host> --vuser <vcenter-username> --vpass <vcenter-password>

	Connecting to vCenter: vcenter.host...done!

	Exploring vCenter resources...done!

	Do you want to process datacenter Datacenter (y/[n])? y

	  * vCenter cluster found:
		  - Name       : Cluster2
		  - Location   : /
		Import cluster (y/[n])? y

		In which OpenNebula cluster do you want the vCenter cluster to be included?


		  - ID: 100 - NAME: Cluster
		  - ID: 101 - NAME: Cluster3

		Specify the ID of the cluster or press Enter if you want OpenNebula to create a new cluster for you:

		OpenNebula host Cluster2 with ID 2 successfully created.

.. note:: If vCenter is using a port other than the default port, you can use the --port command.

Import a vCenter cluster with Sunstone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can also import a cluster from Sunstone. Click on Hosts under the Infrastructure menu entry and then click on the Plus sign, a new window will be opened.

.. image:: /images/vcenter_create_host.png
    :align: center

Select VMWare vCenter from the Type drop-down menu. Introduce the vCenter hostname (the <SERVER>:<PORT> notation can be used for non default ports) or IP address and the credentials used to manage the vCenter instance and click on **Get Clusters**

Once you enter the vCenter credentials you’ll get a list of the vCenter clusters that haven't been imported yet. You’ll have the name of the vCenter cluster and the location of that cluster inside the Hosts and Clusters view in vSphere.

.. note:: A vCenter cluster is considered that it hasn't been imported if the cluster's moref and vCenter instance uuid is not found in OpenNebula's image pool.

If OpenNebula founds new clusters they will be grouped by the datacenter they belong.

.. image:: /images/vcenter_create_host_step2.png
    :align: center

Before you check one or more vCenter clusters to be imported, you can select an OpenNebula cluster from the drop-down Cluster menu, if you select the default datastore (ID:0), OpenNebula will create a new OpenNebula cluster for you.

.. image:: /images/vcenter_create_host_step3.png
    :align: center

Select the vCenter clusters you want to import and finally click on the Import button. Once the import tool finishes you’ll get the ID of the OpenNebula hosts created as representations of the vCenter clusters.

.. image:: /images/vcenter_create_host_step4.png
    :align: center

You can check that the hosts representing the vCenter clusters have a name containing the cluster name, and if there is name collision with a previously imported vCenter cluster, a string is added to avoid the collision. Also you can see that if you select the default datastore, OpenNebula will assign a new OpenNebula cluster with the same name of the imported vCenter cluster.

.. image:: /images/vcenter_create_host_step5.png
    :align: center

Note that if you delete an OpenNebula host representing a vCenter cluster and if you try to import it again you may have an error like the following.

.. image:: /images/vcenter_create_host_step6.png
    :align: center

In that case should specify the right cluster from the Cluster drop-down menu or remove the OpenNebula Cluster so OpenNebula can create the cluster again automatically when the vCenter Cluster is imported.

You can define ``VM_PREFIX`` attribute within host template. This attribute means that when you instanciate a VM in this host, all vm will be named begin with ``VM_PREFIX``.

.. _vcenter_import_resources:

Importing vCenter resources
--------------------------------------------------------------------------------

Once you have imported your vCenter cluster you can import the rest of the vCenter resources delegating the authentication to the imported OpenNebula host.

There are two steps to be followed  to import vCenter resources:

* Retrieve a list of the resources available to identify the concrete ones to import:

    - [CLI]      Using onevcenter list -o <resource type> -h <host_id> [additional_info].
    - [Sunstone] Navigate to the proper section on sunstone and click on import button and select the proper host.

This will show you the list of objects that you can import giving you some information.

* Import selected resources based on the previous information collected by the first step:

    - [CLI]      Using onevcenter import <desired objects> -o <resource type> -h <host_id> [additional_info].

        There are several ways to perform this operation, in this list an ID column arranging the unimported resources will appear in addition to the REF column, you can use both columns to select certain resoures:

        +---------------------------------+-----------------------------------------------------------------------------------+
        |   Command (Example)             | Note                                                                              |
        +---------------------------------+-----------------------------------------------------------------------------------+
        | onevcenter import ref           | This will import the resource with ref                                            |
        +---------------------------------+-----------------------------------------------------------------------------------+
        | onevcenter import 0             | This will import the first resource showd on the list, the resource with IM_ID 0  |
        +---------------------------------+-----------------------------------------------------------------------------------+
        | onevcenter import "ref0, ref1"  | This will import both items with refs ref0 and ref1                               |
        +---------------------------------+-----------------------------------------------------------------------------------+
        | onevcenter import 0..5          | This will import items with IM_ID 0, 1, 2, 3, 4, 5                                |
        +---------------------------------+-----------------------------------------------------------------------------------+

    - [Sunstone] Simply select the desired resources (checking any option) from the previous list and click Import.

Importing all resources with default configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In some scenarios you will want to import every resource using default values avoiding the interactive interface.

- [CLI] using onevcenter import_defaults command:

.. prompt:: bash $ auto

    onevcenter import_defaults -o datastores -h 0

This will import all datastores related to the imported OpenNebula host with ID: 0.

- [Sunstone] Click on the first checkbox at the corner of the table.

.. _vcenter_import_datastores:

Importing vCenter Datastores
--------------------------------------------------------------------------------

Virtual hard disks, which are attached to vCenter virtual machines and templates, have to be represented in OpenNebula as images. Images must be placed in OpenNebula's image datastores which can be easily created thanks to the import tools. vCenter datastores can be imported using the ``onevcenter`` tool or the Sunstone user interface.

A vCenter datastore is unique inside a datacenter, so it is possible that two datastores can be found with the same name in different datacenters and/or vCenter instances. In  this situations, OpenNebula generates a name that avoids collisions. This name can be changed once the datastore has been imported to a more human-friendly name.

Import a datastore with onevcenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here's an example showing how a datastore is imported using the command-line interface:

First of all we already have one vCenter cluster imported with ID 0.

.. prompt:: bash $ auto

    onevcenter list -o datastores -h 0

    # vCenter: vCenter.server

    IMID REF             NAME                                               CLUSTERS
    0    datastore-15    datastore2                                         [102]
    1    datastore-11    datastore1                                         []
    2    datastore-15341 datastore1 (1)                                     [100]
    3    datastore-16    nfs                                                [102, 100]

The import tool (list) will discover datastores in each datacenter and will show the name of the datastore, the capacity and OpenNebula cluster IDs which this datastore will be added to.

Once you know what datastore you want to import:

.. prompt:: bash $ auto

    onevcenter import datastore-16 -o datastores -h 0
    ID: 100
    ID: 101

When you select a datastore, two representations of the same datastore are created in OpenNebula: an IMAGE datastore and a SYSTEM datastore that’s why you can see that two datastores have been created (unless the datastore is a StorageDRS, in that case only a SYSTEM datastore is created.

Import a datastore with Sunstone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In Sunstone, click on Datastores under the Storage menu entry and then click on the Import button, a new window will be opened.

.. image:: /images/vcenter_datastore_import_step1.png
    :align: center

In the new window, choose a cluster to authenticate you into this vCenter instance and click on **Get Datastores**, this will retrieve all the datastore available for import - those that hasn't been imiported yet. If OpenNebula clusters IDs column is empty that means that the import tool could not find an OpenNebula cluster where the datastore can be grouped and you may have to assign it by hand later.

From the list, select the datastore you want to import and finally click on the Import button. YOu'll get a notification with the IDs of the datastores that have been created.

In the datastore list you can check the datastore name. Also between parentheses you can find SYS for a SYSTEM datastore, IMG for an IMAGE datastore or StorDRS for a StorageDRS cluster representation. Datastore name can be changed once the datastore has been imported.

.. _vcenter_import_templates:

Importing vCenter VM Templates
--------------------------------------------------------------------------------

The **onevcenter** tool and the Sunstone interface can be used to import existing VM templates from vCenter.

.. important:: This step should be performed **after** we have succesfully imported the datastores where the template's hard disk files are located, and those datastores have been monitored at least once.

OpenNebula will create OpenNebula images that represents vCenter VM disks, and virtual networks that represents the port groups used by the virtual NICs. For example, we have a template that has three disks and a nic connected to the VM Network port group.

.. image:: /images/vcenter_template_import_step3.png
    :width: 70%
    :align: center

After the import operation finishes there will be three images representing each of the virtual disks found within the template. The name of the images can be changed after the images have been imported.

.. image:: /images/vcenter_template_import_step4.png
    :width: 70%
    :align: center

Also a virtual network will be created. Note that the virtual network is added automatically to an OpenNebula cluster where the vCenter cluster has been added as a host.

A vCenter template name is only unique inside a folder, so you may have two templates with the same name in different folders inside a datacenter. If OpenNebula detects a collision it will craft a name to avoid said collision. This name can be changed after the import finishes.

.. _vcenter_template_import:

Import a VM Template with onevcenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This would be the process using the **onevcenter** tool.

.. prompt:: bash $ auto

    $ onevcenter list -o templates -h 0

    # vCenter: vcenter.Server

    IMID REF        NAME
       0 vm-8720    corelinux7 x86_64 with spaces
       1 vm-9199    one-corelinux7_x86_64
       2 vm-8663    dist_nic_test

In this example our vcenter.server has 3 templates and they are listed from IM_ID = 0 to 2.

Whenever you are ready to import:

.. prompt:: bash $ auto

    onevcenter import vm-1754 -o templates -h 0

    - Template: corelinux7_x86_64

You'll be asked about whether or not you want to use :ref:`linked clones <vcenter_linked_clones_description>`. If a copy of the template is needed, this action may take some time as a full clone of the template and its disks has to be performed.

You can also select the folder where you want VMs based on this template to be shown in vSphere's VMs and Templates inventory.

OpenNebula will inspect the vCenter template and will create images and networks for the virtual disks and virtual networks associated to the template. Those actions will require some time to finish as well.

By default OpenNebula will use the first Resource Pool that is available in the datacenter unless a specific Resource Pool has been set for the host representing the vCenter cluster. If you haven't already have a look to the :ref:`"Resource Pools in OpenNebula" section in this chapter<vcenter_resource_pool>`. If you want to select a new resource pool, a list of available Resource Pools will display so you can select one of them:

.. prompt:: text $ auto

    The list of available resource pools is:

    - TestResourcePool/NestedResourcePool
    - TestResourcePool

    Please input the new default resource pool name:

If you want to create a list of Resource Pools that will allow the user to select one of them, you have the chance of accepting the list generated by the import tool or enter the references of the Resource Pools using a comma to separate the values.

If you selected a list, you will be asked to select the reference of the default Resource Pool in that list.

Import a VM Template with Sunstone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In Sunstone, click on VMs under the Template menu entry and then click on the Import button. In the new window, choose the OpenNebula host representing the vCenter where the VM Template resides.

OpenNebula will search for templates that haven't been imported yet.

.. image:: /images/vcenter_template_import_step8.png
    :width: 50%
    :align: center

Before importing a template, you can click on the down arrow next to the template's name and specify the Resource Pools as it was explained in the :ref:`Resource Pools in OpenNebula section in this chapter <vcenter_resource_pool>`. If the vCenter cluster doesn't have DRS enabled you won't be able to use Resource Pools and hence the down arrow won't display any content at all.

Select the template you want to import and finally click on the Import button. This process may take some time as OpenNebula will import the disks and network interfaces that exists in the template and will create images and networks to represent them.

A vCenter template is considered that it hasn't been imported if the template's moref and vCenter instance uuid is not found in OpenNebula's template pool. If OpenNebula does not find new templates, check that you have previously imported the vCenter clusters that contain those templates.

When an image is created to represent a virtual disk found in the vCenter template, the VCENTER_IMPORTED attribute is set to YES automatically. This attribute prevents OpenNebula to delete the file from the vCenter datastore when the image is deleted from OpenNebula.

.. note:: If you want to use linked clones with a template, please import it using the **onevcenter** tool.

After a vCenter VM Template is imported as a OpenNebula VM Template, it can be modified to change the capacity in terms of CPU and MEMORY, the name, permissions, etc. It can also be enriched to add:

* :ref:`New disks <disk_hotplugging>`
* :ref:`New network interfaces <vm_guide2_nic_hotplugging>`
* :ref:`Context information <vcenter_contextualization>`

.. _vcenter_opennebula_managed:

If you modify a VM template and you edit a disk or nic that was found by OpenNebula when the template was imported, please consider the following:

    * Disks and nics that were discovered in a vCenter template have a special attribute called OPENNEBULA_MANAGED set to NO.
    * The OPENNEBULA_MANAGED=NO should only be present in DISK and NIC elements that exist in your vCenter template as OpenNebula doesnt't apply the same actions that those applied to disks and nics that are not part of your vCenter template.
    * If you edit a DISK or NIC element in your VM template which has OPENNEBULA_MANAGED set to NO and you change the image or virtual network associated to a new resource that is not part of the vCenter template please don't forget to remove the OPENNEBULA_MANAGED attribute in the DISK or NIC section of the VM template either using the Advanced view in Sunstone or from the CLI with the onetemplate update command.

Before using your OpenNebula cloud you may want to read about the :ref:`vCenter specifics <vcenter_specifics>`.

.. _vcenter_import_wild_vms:

Importing running Virtual Machines
--------------------------------------------------------------------------------

Once a vCenter cluster is monitored, OpenNebula will display any existing VM as Wild. These VMs can be imported and managed through OpenNebula once the host has been successfully acquired.

*Requirements*

* **Before** you import a Wild VM you must have imported the datastores where the VM's hard disk files are located as it was explained before. OpenNebula requires the datastores to exist before the image that represents an existing virtual hard disk is created.
* Running VM cannot have snapshots. Please remove them before importing.

In the command line we can list wild VMs with the one host show command:

.. prompt:: text $ auto

    $ onehost show 0
      HOST 0 INFORMATION
      ID                    : 0
      NAME                  : MyvCenterHost
      CLUSTER               : -
      [....]

      WILD VIRTUAL MACHINES

                NAME                                                      IMPORT_ID  CPU     MEMORY
                test-rp-removeme - Cluster                                  vm-2184    1        256

      [....]

In Sunstone we have the Wild tab in the host's information:

.. image:: /images/vcenter_wild_vm_list.png
    :width: 70%
    :align: center

VMs in running state can be imported, and also VMs defined in vCenter that are not in Power On state (this will import the VMs in OpenNebula as in the poweroff state).

.. _vcenter_wild_vm_nic_disc_import:

A Wild VM import process creates images to represent the VM disks as well as new Virtual Networks if they are not already represented. If a Virtual Network exists already in OpenNebula, a network lease (IP/MAC) is requested for each IP reported for the VM by the VMware tools. If no AR contains the IP address space of the IP reported by the VM, a new AR is created and a lease requested. If the same NIC in the vCenter VM reports more than one IP, this is represented using NIC_ALIAS.

It is important to clarify that in the event that a VM Template has multiple NICs and NIC ALIAS, they will be imported during this process.

To import existing VMs you can use the 'onehost importvm' command.

.. prompt:: text $ auto

    $ onehost importvm 0 "test-rp-removeme - Cluster"
    $ onevm list
    ID USER     GROUP    NAME            STAT UCPU    UMEM HOST               TIME
     3 oneadmin oneadmin test-rp-removem runn 0.00     20M [vcenter.v     0d 01h02

Also the Sunstone user interface can be used from the host's Wilds tab. Select a VM from the list and click on the Import button.

.. image:: /images/vcenter_wild_vm_list_import_sunstone.png
    :width: 70%
    :align: center

After a Virtual Machine is imported, their life-cycle (including creation of snapshots) can be controlled through OpenNebula. The following operations *cannot* be performed on an imported VM:

* Recover --recreate
* Undeploy (and Undeploy --hard)
* Stop

Once a Wild VM is imported, OpenNebula will reconfigure the vCenter VM so VNC connections can be established once the VM is monitored.

Also, network management operations are present like the ability to attach/detach network interfaces, as well as capacity (CPU and MEMORY) resizing operations and VNC connections if the ports are opened beforehand.

.. _vcenter_reimport_wild_vms:

After a VM has been imported, it can be removed from OpenNebula and imported again. OpenNebula sets information in the vCenter VM metadata that needs to be removed, which can be done with the ``onevcenter cleartags`` command:

- opennebula.vm.running
- opennebula.vm.id
- opennebula.disk.*

The following procedure is useful if the VM has been changed in vCenter and OpenNebula needs to "rescan" its disks and NICs:

* Use onevcenter cleartags on the VM that will be removed:

.. prompt:: bash $ auto

    $ onevcenter cleartags <vmid>

**vmid** is the id of the VM whose attributes will be cleared.

* Un-register VM

.. prompt:: bash $ auto

    $ onevm recover --delete-db <vmid>

* Re-import VM: on the next host's monitoring cycle you will find this VM under **Wilds** tab, and it can be safely imported.

.. _vcenter_import_networks:

Importing vCenter Networks
--------------------------------------------------------------------------------

OpenNebula can create Virtual Network representations of existing vCenter networks (standard port groups and distributed port groups). OpenNebula can handle on top of these representations three types of Address Ranges: Ethernet, IPv4 and IPv6. This networking information can be passed to the VMs through the contextualization process.

When you import a vCenter port group or distributed port group, OpenNebula will create an OpenNebula Virtual Network that represents that vCenter network.

.. note:: Multicluster networks are supported by OpenNebula, distributed port groups spanning across than 1 vCenter cluster can be properly imported. OpenNebula will show up the related vCenter clusters and at least 1 should be imported before proceeding with the network import process.
          Even if it is possible to import a multicluster network having only 1 vCenter cluster imported, it is best to import all vCenter clusters related to the network into OpenNebula first (arranging them into OpenNebula clusters).

A vCenter network name is unique inside a datacenter, so it is possible that two networks can be found with the same name in different datacenters and/or vCenter instances. If OpenNebula detects a collision it will craft a name to avoid the collision. This name can be changed afterwards.

.. _import_network_onevcenter:

Import networks with onevcenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The import tool will discover port groups in each datacenter and will show the name of the port group, the port group type (Port Group or Distributed Port Group), the cluster that uses that port group and the OpenNebula cluster ID which this virtual network will be added to.

In case that the network had more than 1 vCenter cluster associated, the list command will show a list of the OpenNebula clusters.

Here's an example showing how a standard port group or distributed port group is imported using the command-line interface.

.. prompt:: bash $ auto

    $ onevcenter list -o networks -h 0

    # vCenter: vcenter.Server

	IMID REF              NAME                      CLUSTERS
	0    network-12       VM Network                [100, 102]
	1    network-12245    testing00                 [100, 102]
	2    network-12247    testing03                 [102]
	3    network-12248    testing02                 [102]
	4    network-12246    testing01                 [100, 102]

With this information, we now want to import 'testing0*' networks (it's common to import more than one network).

.. prompt:: bash $ auto

    $ onevcenter import 1..4 -o networks -h 0

or

.. prompt:: bash $ auto

    $ onevcenter import "network-12245, network-12247, network-12246, network-12248" -o networks -h 0

After this you'll be asked several questions and different actions will be taken depending on your answers.

If you want to import the network and the vnet has vlan id it will show to you in first place.
Next step is to assign an Address Range. You can know more about address ranges in the :ref:`Managing Address Ranges <manage_address_ranges>` section.

Import networks with Sunstone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In Sunstone the process is similar, click on Virtual Networks under the Network menu entry and then click on the Import button. Choose a vCenter cluster and then on **Get Networks**.

Before importing a network, you can click on the down arrow next to the network's name and specify the type of address pool you want to configure:

* eth for an Ethernet address range pool.
* ipv4 for an IPv4 address range pool.
* ipv6 for an IPv6 address range pool with SLAAC.
* ipv6_static for an IPv6 address range pool without SLAAC (it requires an IPv6 address and a prefix length).

When you import a network, the default address range is a 255 MAC addresses pool.

Finally click on the Import button, the ID of the virtual network  that has been created will be displayed.

If OpenNebula does not find new networks, check that you have previously imported the vCenter clusters that are using those port groups.

Importing vCenter Images
--------------------------------------------------------------------------------

OpenNebula can create image representations of vCenter VMDK and ISO files that are present in vCenter datastores.

When you import an image, OpenNebula generates a name that avoids collisions, that name contains the image name and, if there was another image with that name, a suffix. That name can be changed once the image has been imported to a more human-friendly name. This is sample name:

The import tools will look for files that haven't been previously imported, checking if there's a file with the same PATH and DATASTORE_ID attributes.

.. _vcenter_import_images:

Import images with onevcenter
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The **onevcenter** tool and the Sunstone interface can be used to import this kind of files.

The onevcenter tool needs that an OpenNebula's IMAGE datastore name is specified as an argument. OpenNebula will browse the datastores and look for VMDK and ISO files. This means that it's mandatory to have the proper vCenter image datastore imported into OpenNebula, we can pass on this information through onevcenter tool with -d option so be sure to check this before the import image operation:

This is an easy way for check available vCenter datastores:

.. prompt:: bash $ auto

	onedatastore list | grep -E 'img.*vcenter'

	 100 datastore2(IM       924G 100%  102               1 img  vcenter vcenter on
	 102 datastore1(IM       924G 88%   -                 0 img  vcenter vcenter on
	 106 nfs(IMG)            4.5T 39%   100,102          24 img  vcenter vcenter on


Here's an example showing how a VMDK file can be imported using the command-line interface.
In this case we are going to use datastore1 (102) and host 0:

.. prompt:: bash $ auto

	onevcenter list -o images -h 0 -d 106
	# vCenter: vcenter.vcenter65-1

	IMID REF                                 PATH
  	   0 one-21                              one_223304/21/one-21.vmdk
	   1 Core-current.iso.iso                one_223304/22/Core-current.iso.iso

Once the image has been imported, it will report the OpenNebula image ID.

Import images with Sunstone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Images can also be imported from Sunstone. Click on Images under the Storage menu entry and click on the Import button. In the new window, choose a OpenNebula host representing the vCenter cluster containing the datastore where the image belongs and click on **Get Images**.

OpenNebula will search for VMDK and ISO files that haven't been imported yet.

.. image:: /images/vcenter_image_import_step3.png
    :width: 50%
    :align: center

Select the images you want to import and click on the Import button. The ID of the imported images will be reported.

.. _vcenterc_image:

When an image is created using the import tool, the VCENTER_IMPORTED attribute is set to YES automatically. This attribute prevents OpenNebula to delete the file from the vCenter datastore when the image is deleted from OpenNebula, so it can be used to prevent a virtual hard disk to be removed accidentally from a vCenter template. This default behavior can be changed in ``/var/lib/one/etc/remotes/vmm/vcenter/vcenterc`` by setting DELETE_IMAGES to Yes.

.. _vcenter_migrate:

Migrate vCenter Virtual Machines with OpenNebula
------------------------------------------------

vCenter Driver allows migration of VMs between different vCenter clusters (ie, OpenNebula hosts) and/or different datastores. Depending on the type of migration (cold, the VM is powered off, or saved; or live, the VM is migrated while running), or the target (cluster and/or datastore), several requirements needs to be met in order to migrate the machine.

Migrating a VM Between vCenter Clusters (OpenNebula Hosts)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Requirements (both live and cold migrations)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Every Network attached to the selected VMs need to exists in both vCenter clusters and OpenNebula clusters
* Every Datastore that is used by the VM need to exist in both vCenter clusters and OpenNebula clusters
* Target OpenNebula host can specify a ESX_MIGRATION_LIST attribute:
    - If not specified, target ESX host is not explicitly declared and migration may fail
    - If set to an empty string (""), OpenNebula will randomly chose a target ESX from all the ESXs that belong to the vCenter target cluster
    - If set to a space-separated list of ESX hostnames (that need to beling to the vCenter target cluster), OpenNebula will randomly chose a target ESX from the list

A good place to check if the VM meets the OpenNebula requirements is to peep into the 'AUTOMATIC_REQUIREMENTS' attribute of the Virtual Machine (this can be reviewed in the Template info tab) and check if it includes the target OpenNebula clusters (remember, a cluster in OpenNebula is a collection of hosts, virtual networks and datastores, a cluster in vCenter is represented as a host in OpenNebula).

Requirements (only live migrations)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* vMotion interface enabled in both vCenter clusters (otherwise the driver will warn about compatibility issues)
* OpenNebula live migration only works for running VMs so be sure to check the state before

Usage (CLI)
^^^^^^^^^^^

**Cold Migration**

.. prompt:: bash $ auto

    $ onevm migrate "<VM name>" <destination host id>

**Live Migration**

.. prompt:: bash $ auto

    $ onevm migrate --live "<VM name>" <destination host id>


Migrating a VM Between Datastores
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On a VM migration, target datastore can be changed. Disks belonging to the VM will be migrated to the target datastore. This is useful for rebalancing resources usage among datastores.

Requirements (both cold and live migrations)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Every Datastore that is used by the VM needs to exist in both vCenter clusters and OpenNebula clusters

Usage (CLI)
^^^^^^^^^^^

**Cold Migration**

.. prompt:: bash $ auto

    $ onevm migrate "<VM name>" <destination host id> <destination datastore id>

**Live Migration**

.. prompt:: bash $ auto

    $ onevm migrate --live "<VM name>" <destination host id> <destination datastore id>

.. _vcenter_hooks:

vCenter Hooks
-------------

OpenNebula has two hooks to manage networks in vCenter and :ref:`NSX <nsx_setup>`.

+----------------------+--------------------------------------------------------+
| Hook Name            | Hook Description                                       |
+======================+========================================================+
| vcenter_net_create   | Allows you to create / import vCenter and NSX networks |
+----------------------+--------------------------------------------------------+
| vcenter_net_delete   | Allows you to delete vCenter and NSX networks          |
+----------------------+--------------------------------------------------------+

These hooks should be created automatically when you add a vCenter cluster. If accidentially deleted, they can be created again manually.

Go to `Create vCenter Hooks`_ and follow the steps to create a new hook.

.. note:: Detailed information about how hooks work is available :ref:`here <hooks>`.


List vCenter Hooks
~~~~~~~~~~~~~~~~~~

Type the next command to list registered hooks:

.. prompt:: bash $ auto

    $ onehook list

The output of the command should be something like this:

.. image:: /images/nsx_hook_list.png


Create vCenter Hooks
~~~~~~~~~~~~~~~~~~~~~

The following command can be used to create a new hook:

.. prompt:: bash $ auto

    $ onehook create <template_file>

where template file is the name of the file that contains the hook template information.

The hook template for network creation is:

.. prompt:: bash $ auto

    NAME = vcenter_net_create
    TYPE = api
    COMMAND = vcenter/create_vcenter_net.rb
    CALL = "one.vn.allocate"
    ARGUMENTS = "$API"
    ARGUMENTS_STDIN = yes

The latest version <https://raw.githubusercontent.com/OpenNebula/one/master/share/hooks/vcenter/templates/create_vcenter_net.tmpl>`__

.. _vcenter_net_delete_template:

The hook template for network deletion is:

.. prompt:: bash $ auto

    NAME = vcenter_net_delete
    TYPE = api
    COMMAND = vcenter/delete_vcenter_net.rb
    CALL = "one.vn.delete"
    ARGUMENTS = "$API"
    ARGUMENTS_STDIN = yes

The latest version of the hook delete template can be found `here <https://raw.githubusercontent.com/OpenNebula/one/master/share/hooks/vcenter/templates/delete_vcenter_net.tmpl>`__

Delete vCenter Hooks
~~~~~~~~~~~~~~~~~~~~

A hook can be deleted if its ID is known. The ID can be retrieved using onehook list, and then deleted using the following command.

.. prompt:: bash $ auto

    $ onehook delete <hook_id>

.. _driver_tuning:

Driver tuning
-------------

Drivers can be easily customized please refer to :ref:`vCenter Driver Section <vcenter_driver>` in the :ref:`Integration Guide <integration_guide>`.

Some aspects of the driver behavior can be configured on */var/lib/one/remotes/etc/vmm/vcenter/vcenterrc*:

* **delete_images**: Allows OpenNebula to delete imported vCenter images. Default: **no**.

* **vm_poweron_wait_default**: Timeout for deploy action. Default: **300**.

* **debug_information**: Provides more verbose logs. Default: **false**.

* **retries**: Some driver actions support a retry if a failure occurs. This parameter will set the amount of retries. Default: **3**.

* **retry_interval**: Amount of time to wait between retry attempts (seconds). Default: **1**.

* **memory_dumps**: Create snapshots with memory dumps. Default: **true**.

* **keep_non_persistent_disks**: Detach non persistent disks from VMs on VM terminate but avoid deleting them afterwards. Default: **false**.
