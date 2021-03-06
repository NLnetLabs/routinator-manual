Installation Notes
==================

In certain scenarios and on some platforms specific steps are needed in order to
get Routinator working as desired.

Enabling or Disabling Features
------------------------------

When you build Routinator yourself using Cargo, `"features"
<https://doc.rust-lang.org/cargo/reference/features.html>`_ provide a mechanism
to express conditional compilation and optional dependencies. The Routinator
package defines a set of named features in the ``[features]`` table of
`Cargo.toml <https://github.com/NLnetLabs/routinator/blob/main/Cargo.toml>`_.
The table also defines if a feature is enabled or disabled by default.

Routinator currently has the following features:

``socks`` —  *Enabled* by default
    Allow the configuration of a SOCKS proxy.
``ui``  —  *Enabled* by default
    Download and build the the `routinator-ui <https://crates.io/crates/routinator-ui>`_ crate to run the :doc:`user interface<user-interface>`.
``native-tls`` —  *Disabled* by default
    Use the native TLS implementation of your system instead of `rustls <https://github.com/rustls/rustls>`_.
``rta`` —  *Disabled* by default
    Let Routinator validate `Resource Tagged Attestations (RTAs) <https://datatracker.ietf.org/doc/html/draft-ietf-sidrops-rpki-rta>`_.
    
To disable the features that are enabled by default, use the
``--no-default-features`` option. You can then choose which features you want
using the ``--features`` option, listing each feature separated by commas. 

For example, if you want to build Routinator without the user interface, make 
sure SOCKS support is retained and use the native TLS implementation, enter the 
following command:

.. code-block:: text

   cargo install --locked --no-default-features --features socks,native-tls routinator

Statically Linked Routinator
----------------------------

While Rust binaries are mostly statically linked, they depend on :program:`libc`
which, as least as :program:`glibc` that is standard on Linux systems, is
somewhat difficult to link statically. This is why Routinator binaries are
actually dynamically linked on :program:`glibc` systems and can only be
transferred between systems with the same :program:`glibc` versions.

However, Rust can build binaries based on the alternative implementation named
:program:`musl` that can easily be statically linked. Building such binaries is
easy with :program:`rustup`. You need to install :program:`musl` and the correct
:program:`musl` target such as ``x86_64-unknown-linux-musl`` for x86\_64 Linux
systems. Then you can just build Routinator for that target.

On a Debian (and presumably Ubuntu) system, enter the following:

.. code-block:: bash

   sudo apt-get install musl-tools
   rustup target add x86_64-unknown-linux-musl
   cargo build --target=x86_64-unknown-linux-musl --release

Using Tmpfs for the RPKI Cache
------------------------------

The full RPKI data set consists of hundreds of thousands of small files. This
causes a considerable amount of disk I/O with each validation run. If this is
undesirable in your setup, you can choose to store the cache in volatile memory
using the `tmpfs file system
<https://www.kernel.org/doc/html/latest/filesystems/tmpfs.html>`_.

When setting this up, you should make sure to only put the directory for the
local RPKI cache in *tmpfs* and not the directory where the Trust Anchor
Locators reside. Both locations are set in the :ref:`configuration file
<manual-page:configuration file>` with the ``repository-dir`` and ``tal-dir``
options, respectively.

If you have installed Routinator using a package, by default the RPKI cache
directory will be :file:`/var/lib/routinator/rpki-cache`, so we'll use that as
an example. Note that the directory you choose must exist before the mount can
be done. You should allocate at least 3GB for the cache, but giving it 4GB will
allow ample margin for future growth:

.. code-block:: bash

    sudo mount -t tmpfs -o size=4G tmpfs /var/lib/routinator/rpki-cache

*Tmpfs* will behave just like a regular disk, so if it runs out of space
Routinator will do a clean crash, stopping validation, the API, HTTP server 
and most importantly the RTR server, ensuring that no stale data will be
served to your routers. 

Also keep in mind that every time you restart the machine, the contents of the
*tmpfs* file system will be lost. This means that Routinator will have to
rebuild its cache from scratch. This is not a problem, other than it having to
download several gigabytes of data, which usually takes about ten minutes to
complete. During this time all services will be unavailble.

Note that your routers should be configured to have a secondary relying party
instance available at all times.

Platform Specific Instructions
------------------------------

.. Tip:: GÉANT has created an
         `Ansible playbook <https://github.com/GEANT/rpki-validation-tools>`_
         defining a role to deploy Routinator on Ubuntu.

For some platforms, :program:`rustup` cannot provide binary releases to install
directly. The `Rust Platform Support
<https://doc.rust-lang.org/nightly/rustc/platform-support.html>`_ page lists
several platforms where official binary releases are not available, but Rust is
still guaranteed to build. For these platforms, automated tests are not run so
it’s not guaranteed to produce a working build, but they often work to quite a
good degree.

OpenBSD
"""""""

On OpenBSD, `patches
<https://github.com/openbsd/ports/tree/master/lang/rust/patches>`_ are required
to get Rust running correctly, but these are well maintained and offer the
latest version of Rust quite quickly.

Rust can be installed on OpenBSD by running:

.. code-block:: bash

   pkg_add rust

CentOS 6
""""""""

The standard installation method does not work when using CentOS 6. Here, you
will end up with a long list of error messages about missing assembler
instructions. This is because the assembler shipped with CentOS 6 is too old.

You can get the necessary version by installing the `Developer Toolset 6
<https://www.softwarecollections.org/en/scls/rhscl/devtoolset-6/>`_ from the
`Software Collections
<https://wiki.centos.org/AdditionalResources/Repositories/SCL>`_ repository. On
a virgin system, you can install Rust using these steps:

.. code-block:: bash

   sudo yum install centos-release-scl
   sudo yum install devtoolset-6
   scl enable devtoolset-6 bash
   curl https://sh.rustup.rs -sSf | sh
   source $HOME/.cargo/env

SELinux using CentOS 7
""""""""""""""""""""""

This guide, contributed by `Rich Compton
<https://github.com/racompton/routinator_centos7_install>`_, describes how to
run Routinator on Security Enhanced Linux (SELinux) using CentOS 7.

1. Start by setting the hostname:

.. code-block:: bash

  sudo nmtui-hostname
  Hostname will be set

2.	Set the interface and connect it:

.. Note:: Ensure that "Automatically connect" and "Available to all users"
          are checked.

.. code-block:: bash

  sudo nmtui-edit

3.	Install the required packages:

.. code-block:: bash

  sudo yum check-update
  sudo yum upgrade -y
  sudo yum install -y epel-release
  sudo yum install -y vim wget curl net-tools lsof bash-completion yum-utils \
      htop nginx httpd-tools tcpdump rust cargo rsync policycoreutils-python

4.	Set the timezone to UTC:

.. code-block:: bash

  sudo timedatectl set-timezone UTC

5.	Remove :program:`postfix` as it is unneeded:

.. code-block:: bash

  sudo systemctl stop postfix
  sudo systemctl disable postfix

6.	Create a self-signed certificate for NGINX:

.. code-block:: bash

  sudo mkdir /etc/ssl/private
  sudo chmod 700 /etc/ssl/private
  sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/nginx-selfsigned.key \
      -out /etc/ssl/certs/nginx-selfsigned.crt
  # Populate the relevant information to generate a self signed certificate
  sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

7.	Add in the :file:`ssl.conf` file to :file:`/etc/nginx/conf.d/ssl.conf` and edit the :file:`ssl.conf` file to provide the IP of the host in the ``server_name`` field.

8.	Replace :file:`/etc/nginx/nginx.conf` with the :file:`nginx.conf` file.

9.	Set the username and password for the web interface authentication:

.. code-block:: bash

  sudo htpasswd -c /etc/nginx/.htpasswd <username>

10.	Start :program:`Nginx` and set it up so it starts at boot:

.. code-block:: bash

  sudo systemctl start nginx
  sudo systemctl enable nginx


11.	Add the user *routinator*, create the :file:`/opt/routinator` directory and assign it to the *routinator* user and group:

.. code-block:: bash

  sudo useradd routinator
  sudo mkdir /opt/routinator
  sudo chown routinator:routinator /opt/routinator

12.	Sudo into the *routinator* user:

.. code-block:: bash

  sudo su - routinator

13.	Install Routinator and add it to the ``$PATH`` for user *routinator*:

.. code-block:: bash

  cargo install --locked routinator
  vi /home/routinator/.bash_profile
  Edit the PATH line to include "/home/routinator/.cargo/bin"
  PATH=$PATH:$HOME/.local/bin:$HOME/bin:/home/routinator/.cargo/bin

14.	Initialise Routinator, accept the ARIN TAL and exit back to the user with :command:`sudo`:

.. code-block:: bash

  /home/routinator/.cargo/bin/routinator -b /opt/routinator init -f --accept-arin-rpa
  exit

15.	Create a routinator systemd script using the template below:

.. code-block:: bash

  sudo vi /etc/systemd/system/routinator.service
  [Unit]
  Description=Routinator RPKI Validator and RTR Server
  After=network.target
  [Service]
  Type=simple
  User=routinator
  Group=routinator
  Restart=on-failure
  RestartSec=90
  ExecStart=/home/routinator/.cargo/bin/routinator -v -b /opt/routinator server \
      --http 127.0.0.1:8080 --rtr <IPv4 IP>:8323 --rtr [<IPv6 IP>]:8323
  TimeoutStartSec=0
  [Install]
  WantedBy=default.target

.. Note:: You must populate the IPv4 and IPv6 addresses. In addition, the IPv6
          address needs to have brackets '[ ]' around it. For example:

          .. code-block:: bash

            /home/routinator/.cargo/bin/routinator -v -b /opt/routinator server \
            --http 127.0.0.1:8080 --rtr 172.16.47.235:8323 --rtr [2001:db8::43]:8323

16.	Configure SELinux to allow connections to localhost and to allow :program:`rsync` to write to the ``/opt/routinator`` directory:

.. code-block:: bash

  sudo setsebool -P httpd_can_network_connect 1
  sudo semanage permissive -a rsync_t

17.	Reload the systemd daemon and set the routinator service to start at boot:

.. code-block:: bash

  sudo systemctl daemon-reload
  sudo systemctl enable routinator.service
  sudo systemctl start routinator.service

18.	Set up the firewall to permit :program:`ssh`, HTTPS and port 8323 for the RTR protocol:

.. code-block:: bash

  sudo firewall-cmd --permanent --remove-service=ssh --zone=public
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" \
      source address="<IPv4 management subnet>" service name=ssh accept'
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" \
      source address="<IPv6 management subnet>" service name=ssh accept'
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" \
      source address="<IPv4 management subnet>" service name=https accept'
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" \
      source address="<IPv6 management subnet>" service name=https accept'
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv4" \
      source address="<peering router IPv4 loopback subnet>" port port=8323 protocol=tcp accept'
  sudo firewall-cmd --permanent --zone public --add-rich-rule='rule family="ipv6" \
      source address="<peering router IPv6 loopback subnet>" port port=8323 protocol=tcp accept'
  sudo firewall-cmd --reload

19. Navigate to :samp:`https://{<IP-address>}/metrics` to see if it's working. You should authenticate with the username and password that you provided in step 10 of setting up the RPKI Validation Server.
