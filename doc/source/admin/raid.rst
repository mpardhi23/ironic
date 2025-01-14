.. _raid:

==================
RAID Configuration
==================

Overview
========
Ironic supports RAID configuration for bare metal nodes.  It allows operators
to specify the desired RAID configuration via the OpenStackClient CLI or REST
API.  The desired RAID configuration is applied on the bare metal during manual
cleaning.

The examples described here use the OpenStackClient CLI; please see the
`REST API reference <https://developer.openstack.org/api-ref/baremetal/>`_
for their corresponding REST API requests.

Prerequisites
=============
The bare metal node needs to use a hardware type that supports RAID
configuration. RAID interfaces may implement RAID configuration either in-band
or out-of-band.

In-band RAID configuration is done using the Ironic Python Agent
ramdisk. For in-band RAID configuration using agent ramdisk, a hardware
manager which supports RAID should be bundled with the ramdisk.
Whether a node supports RAID configuration could be found using the CLI
command ``openstack baremetal node validate <node-uuid>``.

Build agent ramdisk which supports RAID configuration
=====================================================

For doing in-band hardware RAID configuration, Ironic needs an agent ramdisk
bundled with a hardware manager which supports RAID configuration for your
hardware. For example, the :ref:`DIB_raid_support` should be used for HPE
Proliant Servers.

.. note::
    For in-band software RAID, the agent ramdisk does not need to be bundled
    with a hardware manager as the generic hardware manager in the Ironic
    Python Agent already provides (basic) support for software RAID.

RAID configuration JSON format
==============================
The desired RAID configuration and current RAID configuration are represented
in JSON format.

Target RAID configuration
-------------------------
This is the desired RAID configuration on the bare metal node.  Using the
OpenStackClient CLI (or REST API), the operator sets ``target_raid_config``
field of the node. The target RAID configuration will be applied during manual
cleaning.

Target RAID configuration is a dictionary having ``logical_disks``
as the key. The value for the ``logical_disks`` is a list of JSON
dictionaries. It looks like::

  {
   "logical_disks": [
                     {<desired properties of logical disk 1>},
                     {<desired properties of logical disk 2>},
                     .
                     .
                     .
                    ]
  }

If the ``target_raid_config`` is an empty dictionary, it unsets the value of
``target_raid_config`` if the value was set with previous RAID configuration
done on the node.

Each dictionary of logical disk contains the desired properties of logical
disk supported by the hardware type. These properties are discoverable by::

    openstack baremetal --os-baremetal-api-version 1.15 driver raid property list <driver name>

The RAID feature is available in ironic API version 1.15 and above.
If ``--os-baremetal-api-version`` is not used in the CLI, it will error out
with the following message::

   No API version was specified and the requested operation was not
   supported by the client's negotiated API version 1.9. Supported
   version range is: 1.1 to ...

 where the "..." in above error message would be the maximum version
 supported by the service.

The RAID properties can be split into 4 different types:

#. Mandatory properties. These properties must be specified for each logical
   disk and have no default values.

   - ``size_gb`` - Size (Integer) of the logical disk to be created in GiB.
     ``MAX`` may be specified if the logical disk should use all of the
     remaining space available. This can be used only when backing physical
     disks are specified (see below).

   - ``raid_level`` - RAID level for the logical disk. Ironic supports the
     following RAID levels: 0, 1, 2, 5, 6, 1+0, 5+0, 6+0.

#. Optional properties. These properties have default values and
   they may be overridden in the specification of any logical disk.

   - ``volume_name`` - Name of the volume. Should be unique within the Node.
     If not specified, volume name will be auto-generated.

   - ``is_root_volume`` - Set to ``true`` if this is the root volume. At
     most one logical disk can have this set to ``true``; the other
     logical disks must have this set to ``false``. The
     ``root device hint`` will be saved, if the RAID interface is capable of
     retrieving it. This is ``false`` by default.

#. Backing physical disk hints. These hints are specified for each logical
   disk to let Ironic find the desired disks for RAID configuration. This is
   machine-independent information.  This serves the use-case where the
   operator doesn't want to provide individual details for each bare metal
   node.

   - ``share_physical_disks`` - Set to ``true`` if this logical disk can
     share physical disks with other logical disks. The default value is
     ``false``.

   - ``disk_type`` - ``hdd`` or ``ssd``. If this is not specified, disk type
     will not be a criterion to find backing physical disks.

   - ``interface_type`` - ``sata`` or ``scsi`` or ``sas``. If this is not
     specified, interface type will not be a criterion to
     find backing physical disks.

   - ``number_of_physical_disks`` - Integer, number of disks to use for the
     logical disk. Defaults to minimum number of disks required for the
     particular RAID level.

#. Backing physical disks. These are the actual machine-dependent
   information. This is suitable for environments where the operator wants
   to automate the selection of physical disks with a 3rd-party tool based
   on a wider range of attributes (eg. S.M.A.R.T. status, physical location).
   The values for these properties are hardware dependent.

   - ``controller`` - The name of the controller as read by the RAID interface.
     In order to trigger the setup of a Software RAID via the Ironic Python
     Agent, the value of this property needs to be set to ``software``.
   - ``physical_disks`` - A list of physical disks to use as read by the
     RAID interface.

.. note::
    If properties from both "Backing physical disk hints" or
    "Backing physical disks" are specified, they should be consistent with
    each other.  If they are not consistent, then the RAID configuration
    will fail (because the appropriate backing physical disks could
    not be found).

.. note::
    For software RAID as provided by the generic hardware manager that ships
    with the Ironic Python Agent, only the mandatory properties (plus the
    required ``controller`` property) are currently supported.

Examples for ``target_raid_config``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Example 1*. Single RAID disk of RAID level 5 with all of the space
available. Make this the root volume to which Ironic deploys the image::

  {
   "logical_disks": [
                     {
                      "size_gb": "MAX",
                      "raid_level": "5",
                      "is_root_volume": true
                     }
                    ]
  }

*Example 2*. Two RAID disks. One with RAID level 5 of 100 GiB and make it
root volume and use SSD.  Another with RAID level 1 of 500 GiB and use
HDD::

  {
   "logical_disks": [
                     {
                      "size_gb": 100,
                      "raid_level": "5",
                      "is_root_volume": true,
                      "disk_type": "ssd"
                     },
                     {
                      "size_gb": 500,
                      "raid_level": "1",
                      "disk_type": "hdd"
                     }
                    ]
  }

*Example 3*. Single RAID disk. I know which disks and controller to use::

  {
   "logical_disks": [
                     {
                      "size_gb": 100,
                      "raid_level": "5",
                      "controller": "Smart Array P822 in Slot 3",
                      "physical_disks": ["6I:1:5", "6I:1:6", "6I:1:7"],
                      "is_root_volume": true
                     }
                    ]
  }

*Example 4*. Using backing physical disks::

  {
    "logical_disks":
      [
        {
          "size_gb": 50,
          "raid_level": "1+0",
          "controller": "RAID.Integrated.1-1",
          "volume_name": "root_volume",
          "is_root_volume": true,
          "physical_disks": [
                             "Disk.Bay.0:Encl.Int.0-1:RAID.Integrated.1-1",
                             "Disk.Bay.1:Encl.Int.0-1:RAID.Integrated.1-1"
                            ]
        },
        {
          "size_gb": 100,
          "raid_level": "5",
          "controller": "RAID.Integrated.1-1",
          "volume_name": "data_volume",
          "physical_disks": [
                             "Disk.Bay.2:Encl.Int.0-1:RAID.Integrated.1-1",
                             "Disk.Bay.3:Encl.Int.0-1:RAID.Integrated.1-1",
                             "Disk.Bay.4:Encl.Int.0-1:RAID.Integrated.1-1"
                            ]
        }
      ]
  }

*Example 5*. Software RAID with two RAID devices::

  {
   "logical_disks": [
                     {
                      "size_gb": 100,
                      "raid_level": "1",
                      "controller": "software"
                     },
                     {
                      "size_gb": "MAX",
                      "raid_level": "0",
                      "controller": "software"
                     }
                    ]
  }

Current RAID configuration
--------------------------
After target RAID configuration is applied on the bare metal node, Ironic
populates the current RAID configuration.  This is populated in the
``raid_config`` field in the Ironic node. This contains the details about
every logical disk after they were created on the bare metal node. It
contains details like RAID controller used, the backing physical disks used,
WWN of each logical disk, etc. It also contains information about each
physical disk found on the bare metal node.

To get the current RAID configuration::

    openstack baremetal --os-baremetal-api-version 1.15 node show <node-uuid-or-name>

Workflow
========

* Operator configures the bare metal node with a hardware type that has
  a ``RAIDInterface`` other than ``no-raid``. For instance, for Software RAID,
  this would be ``agent``.

* For in-band RAID configuration, operator builds an agent ramdisk which
  supports RAID configuration by bundling the hardware manager with the
  ramdisk. See `Build agent ramdisk which supports RAID configuration`_ for
  more information.

* Operator prepares the desired target RAID configuration as mentioned in
  `Target RAID configuration`_. The target RAID configuration is set on
  the Ironic node::

      openstack baremetal node set <node-uuid-or-name> \
         --target-raid-config <JSON file containing target RAID configuration>

    The CLI command can accept the input from standard input also:
       openstack baremetal node set <node-uuid-or-name> \
          --target-raid-config -

* Create a JSON file with the RAID clean steps for manual cleaning. Add other
  clean steps as desired::


    [{
      "interface": "raid",
      "step": "delete_configuration"
    },
    {
      "interface": "raid",
      "step": "create_configuration"
    }]

  .. note::
    'create_configuration' doesn't remove existing disks.  It is recommended
    to add 'delete_configuration' before 'create_configuration' to make
    sure that only the desired logical disks exist in the system after
    manual cleaning.

* Bring the node to ``manageable`` state and do a ``clean`` action to start
  cleaning on the node::

      openstack baremetal node clean <node-uuid-or-name> \
         --clean-steps <JSON file containing clean steps created above>

* After manual cleaning is complete, the current RAID configuration is
  reported in the ``raid_config`` field when running::

      openstack baremetal node show <node-uuid-or-name>

Limitations of Software RAID
============================

There are certain limitations to be aware of when setting up a Software RAID via the
Ironic Python Agent:

* There is no way to select the disks which are used to set up the software RAID,
  so the Ironic Python Agent will use all available disks. This seems appropriate
  for servers with 2 or 4 disks, but needs to be considered when disk arrays are
  attached.

* The number of created Software RAID devices must be 1 or 2. If there is only one
  Software RAID device, it has to be a RAID-1. If there are two, the first one has
  to be a RAID-1, while the RAID level for the second one can 0, 1, or 1+0. As the
  first RAID device will be the deployment device, enforcing a RAID-1 reduces the
  risk of ending up with a non-booting node in case of a disk failure.

* There is no support for partition images, only whole-disk images are supported with
  Software RAID.

Using RAID in nova flavor for scheduling
========================================

The operator can specify the `raid_level` capability in nova flavor for node to be selected
for scheduling::

  nova flavor-key my-baremetal-flavor set capabilities:raid_level="1+0"

Developer documentation
=======================
In-band RAID configuration is done using IPA ramdisk. IPA ramdisk has
support for pluggable hardware managers which can be used to extend the
functionality offered by IPA ramdisk using stevedore plugins.  For more
information, see Ironic Python Agent `Hardware Manager`_ documentation.

.. _`Hardware Manager`: https://docs.openstack.org/ironic-python-agent/latest/install/index.html#hardware-managers

The hardware manager that supports RAID configuration should do the following:

#. Implement a method named ``create_configuration``. This method creates
   the RAID configuration as given in ``target_raid_config``. After successful
   RAID configuration, it returns the current RAID configuration information
   which ironic uses to set ``node.raid_config``.

#. Implement a method named ``delete_configuration``. This method deletes
   all the RAID disks on the bare metal.

#. Return these two clean steps in ``get_clean_steps`` method with priority
   as 0. Example::

        return [{'step': 'create_configuration',
                 'interface': 'raid',
                 'priority': 0},
                {'step': 'delete_configuration',
                 'interface': 'raid',
                 'priority': 0}]

