
**install ovs**
=============

build ovs
---------

if meet "LT_INIT" error, install libtool, note to use `--prefix=/usr` to install it into `/usr/bin` instead of `/usr/local/bin`.

run these commands to build ovs,

    $ cd ovs
    $ ./boot.sh
    $ ./configure --with-linux=/lib/modules/`uname -r`/build
    $ make
    $ sudo make install
    $ sudo make modules_install

setup database
--------------

    $ ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema

startup ovs
-----------

    $ ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
                         --remote=db:Open_vSwitch,Open_vSwitch,manager_options \
                         --private-key=db:Open_vSwitch,SSL,private_key \
                         --certificate=db:Open_vSwitch,SSL,certificate \
                         --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert \
                         --pidfile --detach
    $ ovs-vsctl --no-wait init
    $ ovs-vswitchd --pidfile --detach

**using ovs**
=============

adding deleting bridges
-----------------------

    ```bash
    $ ovs-vsctl add-br <bridge>
    $ ovs-vsctl list-br
    $ ovs-vsctl list-ports

    $ ovs-vsctl <table> # <table> can be any db's table, like root table 'Open_vSwitch'
    ```

`table` can be any db's table, like root table 'Open_vSwitch', 'Bridge', 'Port', 'Interface'. See ovs-vswitchd.conf.db(5) for help.

run `ovsdb-tool show-log [-mmmm] <file>` to show the log. `-m` means more, use `-mmm` for more info. see `ovsdb-tool --help` for help.

show logs
---------

    ```bash
    $ ovsdb-tool show-log -m
    record 0: "Open_vSwitch" schema, version="7.12.1", cksum="2211824403 22535"

    record 1: 2015-11-17 14:31:55.892 "ovs-vsctl: ovs-vsctl --no-wait init"
            table Open_vSwitch insert row f46de487:

    record 2: 2015-11-17 14:32:13.905
            table Open_vSwitch row f46de487 (f46de487):
    ```

> see `install.md` for detail. `cd ovs && find -name '\*.md'` for docs!!
