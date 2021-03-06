


# Cloud-init and Datasources

## **Cloud-init**

In a cloud environment, we use images to install systems. The system automation is generally done by cloud-init. Cloud-init was originally developed for Ubuntu GNU/Linux on the Amazon EC2 cloud. It has become the de facto installation configuration tool for most Unix-like systems on most cloud environments.

Cloud-init is a tool used to perform initial setup on cloud nodes, including networking, SSH keys, timezone, user data injection, and more. It is a service that runs on the guest, and supports various Linux distributions including Fedora, RHEL, and Ubuntu, even Windows on the latest versions.

Cloud-init is the automated initialization of a new instance which is a task that needs to be split between the cloud infrastructure and the guest OS:

- Hypervisor provides the required metadata via HTTP or via ConfigDrive
- Cloud-init takes care of configuring the instance on Linux.

Cloud-Init has different providers known as **datasources**, which are sources of configuration data for Cloud-init that typically come from the user (**user-data**) or come from the stack that created the configuration drive (**meta-data**). Typical userdata would include files, yaml, and shell scripts while typical metadata would include server name, instance id, display name and other cloud specific details.

For more information, refer to Cloudinit User-Data Formats:

- https://cloudinit.readthedocs.io/en/latest/topics/format.html

<br>

## User-data

user-data must be one of the following types:

- gzip compressed content
- Mime Multi-part archive

Using a mime-multi part file, the user can specify more than one type of data. For example, both a user data script and a cloud-config type could be specified. Supported content-types are listed from the cloud-init subcommand make-mime:

```bash
# cloud-init devel make-mime --list-types
cloud-boothook
cloud-config
cloud-config-archive
cloud-config-jsonp
jinja2
part-handler
upstart-job
x-include-once-url
x-include-url
x-shellscript
```

The bootstrap file generated by vManage is user-data for cloud-init in a Mime Multi-part file. That user-data file contains:

- \#cloud-config section
- #cloud-boothook section

<br>

## Datasources

Datasources are sources of configuration data for cloud-init that typically come from the user (user_data) or come from the stack that created the configuration drive (metadata). 

Typical userdata would include files, yaml, and shell scripts while typical metadata would include server name, instance id, display name and other cloud specific details. 

There are multiple ways to provide this data (each cloud solution seems to prefer its own way).

Known sources:

- [Amazon EC2](https://cloudinit.readthedocs.io/en/latest/topics/datasources/ec2.html)
- Azure
- [Config Drive](https://cloudinit.readthedocs.io/en/latest/topics/datasources/configdrive.html)
- NoCloud
- OpenStack
- Etc 

<br>

### Amazon EC2

The EC2 datasource is the oldest and most widely used datasource that cloud-init supports. This datasource interacts with a *magic* ip that is provided to the instance by the cloud provider. Typically this ip is `169.254.169.254` of which at this ip a http server is provided to the instance so that the instance can make calls to get instance userdata and instance metadata.

### nocloud

The data source `NoCloud` allows the user to provide user-data and meta-data to the instance without running a network service (or even without having a network at all).

You can provide meta-data and user-data to a local vm boot via files on a [vfat](https://en.wikipedia.org/wiki/File_Allocation_Table) or [iso9660](https://en.wikipedia.org/wiki/ISO_9660) filesystem. The filesystem volume label must be `cidata` or `CIDATA`.

These user-data and meta-data files are expected to be in the following format.

```
/user-data
/meta-data
```

Basically, user-data is simply user-data and meta-data is a yaml formatted file representing what you???d find in the EC2 metadata service.

You may also optionally provide a vendor-data file in the following format.

```
/vendor-data
```

This is the option used to instantiate a vEdgeCloud on KVM in this paper.

### Config-Drive

You can use the `config-drive` parameter to present a read-only drive to your instances. This drive can contain selected files that are then accessible to the instance. The configuration drive is attached to the instance at boot, and is presented to the instance as a partition. Configuration drives are useful when combined with *cloud-init* (for server bootstrapping), and when you want to pass large files to your instances.

You can **configure OpenStack** to write metadata to a special **configuration drive** that attaches to the instance when it boots. The instance can mount this **drive** and read files from it to get information that is normally available through the metadata service. This metadata is different from the user data.

The Configuration drive feature enables administrators to automate the configuration of a VM at first boot by creating an ISO disk with user data files. Then Cloud-init or a similar system can configure the VM from the user data. The Configuration drive feature can be used on VMs with private network or no network connectivity, because the VM pulls its own configuration from the ISO and there is no need for an external process to connect to the VM to configure it.

This is the option used to instantiate a CSR1000v on KVM in this paper.

<br>

