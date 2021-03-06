Change log for the StorPool OpenStack integration
=================================================

1.5.0
-----

- add the "groups" command to the `sp-openstack` tool to only check, create,
  and set up the "spopenstack" group and the service accounts' membership,
  as well as the `/var/spool/storpool-spopenstack/` directory

1.4.0
-----

- Note that the StorPool drivers have been included in the Queens release.
- Detect the Queens release of OpenStack and (hopefully) just say that
  the StorPool integration is installed already.

1.3.0
-----

- Add the Pike Cinder, Nova, and os-brick drivers.
- Properly capitalize the 1.2.0 changelog entry.
- Add the `sp-image-to-volume` tool to save a Glance image to a StorPool volume
  and its manual page.

1.2.0
-----

- Add the `-T txn-module` option for use with the txn-install tool.

1.1.1
-----

- Add the sp-openstack.1 manual page.
- Look for the Python modules path in a way compatible with Python 3.

1.1.0
-----

- Add the Ocata Cinder volume driver and Nova attachment driver.
- Add the Newton and Ocata os-brick connector driver.
- Add the capability to make different changes to the same file for
  different OpenStack releases.
- Remove the "probably unaligned" warnings from the documentation of
  the "storpool volume list" and "storpool volume status" checks, since
  the StorPool CLI tool aligns the output in recent versions.
- Let the documentation use the "openstack" client tool where possible.

1.0.0
-----

- Update the Mitaka os-brick connector.
- Update the Liberty and Mitaka Cinder volume drivers.
- Fix the detection of Nova Liberty vs Mitaka.
- Drop support for the Juno and Kilo releases of OpenStack.

0.2.0
-----

- Allow the owner and group of the /var/spool/openstack-storpool/
  shared state directory to be overridden using the -u and -g
  options of the sp-openstack tool.

0.1.0
-----

- Initial public release.
