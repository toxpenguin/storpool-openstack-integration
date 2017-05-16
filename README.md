StorPool Integration with OpenStack
===================================

This document describes the steps necessary to set up an OpenStack
cluster that uses a StorPool distributed storage cluster as a backend
for the Cinder volumes and the Nova instance disks.

Some of the tools referenced here may be found in the [StorPool
OpenStack Integration][github] Git repository.


Preliminary setup
-----------------

1. Set up an OpenStack cluster.

2. Set up a StorPool cluster.

3. Set up at least the StorPool client (the `storpool_beacon` and
   `storpool_block` services) on the OpenStack controller node and on
   each of the hypervisor nodes.

4. On the controller and all the hypervisor nodes, install the
   [`storpool`][py-storpool] and
   [`storpool-spopenstack`][py-spopenstack] Python packages:

        # Install pip (the Python package installer) first
        yum install python-pip
        # ...or...
        apt-get install --no-install-recommends python-pip
        
        pip install storpool
        pip install storpool.spopenstack

5. On the controller and all the hypervisor nodes, clone
   the [StorPool OpenStack Integration][github] Git repository
   or copy it from some other host where it has been cloned:

        # Clone the StorPool OpenStack Integration repository
        git clone https://github.com/storpool/storpool-openstack-integration.git

Set up the Cinder volume backend
--------------------------------

0. Change into the `storpool-openstack-integration/` directory prepared in
   the last step of the "Preliminary setup" section above.

1. Let the StorPool OpenStack integration suite find your Cinder drivers and
   apply the StorPool integration changes:
   and make the necessary modifications in its own work directory:

        ./sp-openstack check
        ./sp-openstack install cinder os_brick

2. Configure Cinder to use the StorPool volume backend.  Edit the
   `/etc/cinder/cinder.conf` file and append the following lines to it:

        [storpool]
        volume_driver=cinder.volume.drivers.storpool.StorPoolDriver
        storpool_template=hybrid

   Then comment out any existing `enabled_backends` or `volume_backend`
   lines and add the following to the `[DEFAULT]` section:

        enabled_backends=storpool

3. Restart the Cinder services on the controller node:

        service cinder-volume restart
        service cinder-scheduler restart
        service cinder-api restart

4. Try to create a Cinder volume:

        cinder create --display-name vol1 1
        cinder list
        cinder extend vol1 2
        cinder list
        
        # Now look for the volume in the StorPool volume list:
        #
        storpool volume list
        
        # Display info about this particular volume in the StorPool list of
        # volumes, obtaining the internal Cinder volume ID from the output of
        # "cinder show".
        #
        storpool volume list | fgrep -e `cinder show vol1 | awk '$2 == "id" { print $4 }'` -e placeTail

Create a Cinder volume based on a Glance image
----------------------------------------------

        # Obtain the internal Glance ID of the "centos 7" image:
        glance image-list | awk '/centos 7/ { print $2 }'
        
        # Now create a volume based on this image:
        cinder create --image-id fdd6ecdd-074c-40fe-8370-0b4438d8c060 --display-name centos-1 3
        
        # Wait for the volume to get out of the "downloading" state:
        cinder list
        
        # In the meantime, monitor the progress in the StorPool volume status
        # display:
        cinder show centos-1 | awk '$2 == "id" { print $4 }'
        storpool volume list | fgrep -e fbca1e54-6901-43af-828f-f2434e8d0bc3
        storpool volume os--volume-fbca1e54-6901-43af-828f-f2434e8d0bc3 status
        
        # Hopefully the volume has moved to the "available" state and is now
        # useable by Nova.

Fill in a Cinder volume directly using `dd` or `qemu-img`
---------------------------------------------------------

Until the StorPool integration with Glance is complete, it might be
easier to fill in a Cinder volume directly from an OS disk image.  These
are the steps needed to do it "manually", attaching the StorPool volume
as a block device to the OpenStack controller node and writing the data
to it.

        mkdir -p spopenstack/img/
        cd spopenstack/img/
        wget -ct0 http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
        cd -
        
        # Check the size needed for the volume:
        qemu-img info spopenstack/img/CentOS-7-x86_64-GenericCloud.qcow2
        
        # Create the volume
        # Note the "id" in the output of "cinder create" and use it further
        # down...
        #
        cinder create --display-name=centos-7-base 8
        
        # Find the StorPool volume with the name corresponding to this
        # internal ID.  It will usually be in the form os--volume-ID, but,
        # just in case, look for it...
        #
        storpool volume list | fgrep -e b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd
        
        # Attach the StorPool volume to our host in read/write mode (default):
        storpool attach here volume os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd
        
        # Make sure the volume is there:
        file -L /dev/storpool/os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd
        
        # Now copy the image (in dd(1) style, but using qemu-img(1)):
        qemu-img convert -n -O raw spopenstack/img/CentOS-7-x86_64-GenericCloud.qcow2 /dev/storpool/os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd
        
        # Very soon after starting the copy operation, make sure that we are
        # trying to write the correct data - a bootable image would most
        # probably start with a boot sector, so the following command should
        # output something like "x86 boot sector":
        #
        file -sL /dev/storpool/os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd
        
        # During the copying process, check the progress using StorPool's
        # "volume status" display:
        #
        storpool volume os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd status
        
        # Now detach the StorPool volume from the controller host:
        # NB: ONLY do this if you are sure that nothing else needs the volume!
        # In this case, we may be sure since we attached it ourselves, but
        # in general, attaching and detaching StorPool volumes by hand on
        # an operational OpenStack installation should only be done rarely and
        # with great care.  If we were using Glance as a data source,
        # there would have been no need for that, since the StorPool Cinder
        # driver would have taken care of attaching and detaching the
        # volume during the "cinder create --image-id ..." operation.
        #
        storpool detach here volume os--volume-b3fb268c-2fb9-4017-a28f-ab33f7dfa0cd

Verify the operation of volume snapshots
----------------------------------------

Make sure that Cinder can create snapshots of existing volumes and use them:

        cinder snapshot-create --display-name centos-7-20150323 centos-7-base
        
        cinder snapshot-list
        storpool snapshot list | fgrep -e 8c4e7260-f7bb-4ff7-9a61-6b3edb692a9a
        
        # Now create a volume from this snapshot:
        cinder create --snapshot-id 8c4e7260-f7bb-4ff7-9a61-6b3edb692a9a --display-name centos-01 8
        cinder list
        
        # And now the following command may display a somewhat surprising
        # result: the snapshot will have 100% allocated disk space, while both
        # the volume that it was created from and the volume that was created
        # from it will have 0% disk space.  This is due to the internal
        # StorPool management of disk data, it should not be a cause for
        # concern.  The important thing is that the last line, the totals,
        # shows that the cluster only has 8 GB "virtual" space and 25 GB
        # "physical" space (approx. 3 * 8 GB due to the replication).
        # Note that the on-disk size may not always be 3 * virtual size, since
        # data may be written continuously, background aggregation may be in
        # progress or may be delayed for a moment when there will be not too
        # many write requests by actual applications, etc.
        #
        storpool volume status

Set up the Nova volume attachment driver (on each hypevisor node)
-----------------------------------------------------------------

0. Change into the `storpool-openstack-integration/` directory prepared in
   the last step of the "Preliminary setup" section above.

1. Make sure that the Python modules [`storpool`][py-storpool] and
   [`storpool.spopenstack`][py-spopenstack]
   have also been installed on all the hypervisor nodes (see the "Preliminary
   setup" section above).

2. Let the StorPool OpenStack integration suite find your Nova drivers and
   and make the necessary modifications in its own work directory:

        ./sp-openstack check
        ./sp-openstack install nova

3. Edit the `/etc/nova/nova.conf` file; comment out any existing
   `volume_drivers` lines and replace them with or add the following one:

        volume_drivers=storpool=nova.virt.libvirt.volume.LibvirtStorPoolVolumeDriver

4. Restart the Nova compute and metadata-api services:

        service nova-compute restart
        service nova-api-metadata restart

Create a Nova instance off a Cinder volume
------------------------------------------

Create a Nova virtual machine and specify that it should boot from a new
volume based on a Cinder snapshot (if done through the Horizon
dashboard, this would be the "Launch Instance >> Instance Boot Source >>
Boot from volume snapshot (creates a new volume)" option):

        nova boot --flavor m1.small --snapshot 8c4e7260-f7bb-4ff7-9a61-6b3edb692a9a --nic net-id=b0be4a34-a665-4953-b7fb-0e8bb84d498c centos-01

[github]: https://github.com/storpool/storpool-openstack-integration
[py-storpool]: https://github.com/storpool/python-storpool
[py-spopenstack]: https://github.com/storpool/python-storpool-spopenstack
[rdo]: https://www.rdoproject.org/
