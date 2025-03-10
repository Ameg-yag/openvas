INSTALLATION INSTRUCTIONS FOR OPENVAS
=====================================

Please note: The reference system used by most of the developers is Debian
GNU/Linux 'Stretch' 9. The build might fail on any other system. Also, it is
necessary to install dependent development packages.

Prerequisites for openvas
-------------------------

Prerequisites:
* a C compiler (e.g. gcc)
* cmake >= 3.0
* libgvm_base, libgvm_util >= 1.0.0
* glib-2.0 >= 2.42
* gio-2.0
* bison
* flex
* libgcrypt >= 1.6
* pkg-config
* libpcap
* libgpgme >= 1.1.2
* redis >= 3.2.0
* libssh >= 0.6.0
* libksba >= 1.0.7
* libgnutls >= 3.2.15

Prerequisites for building documentation:
* Doxygen
* xmltoman (optional, for building man page)
* sqlfairy (optional, for producing database diagram)

Recommended to have WMI support:
* openvas-smb >= 1.0.1

Recommended for extended Windows support (e.g. automatically start the remote registry service):
* impacket-wmiexec of python-impacket >= 0.9.15 found within your PATH

Recommended to have improved SNMP support:
* netsnmp

Install prerequisites on Debian GNU/Linux 'Stretch' 9:

    apt-get install gcc pkg-config libssh-gcrypt-dev libgnutls28-dev libglib2.0-dev \
    libpcap-dev libgpgme-dev bison libksba-dev libsnmp-dev libgcrypt20-dev


Compiling openvas
-----------------

If you have installed required libraries to a non-standard location, remember to
set the `PKG_CONFIG_PATH` environment variable to the location of you pkg-config
files before configuring:

    export PKG_CONFIG_PATH=/your/location/lib/pkgconfig:$PKG_CONFIG_PATH

Create a build directory and change into it with:

    mkdir build
    cd build

Then configure the build with:

    cmake -DCMAKE_INSTALL_PREFIX=/path/to/your/installation ..

Or (if you want to use the default installation path /usr/local):

    cmake ..

This only needs to be done once.

Thereafter, the following commands are useful:

    make                # build the scanner
    make doc            # build the documentation
    make doc-full       # build more developer-oriented documentation
    make install        # install the build
    make rebuild_cache  # rebuild the cmake cache

Please note that you may have to execute `make install` as root, especially if
you have specified a prefix for which your user does not have full permissions.

To clean up the build environment, simply remove the contents of the `build`
directory you created above.


Configuration Options
---------------------

During compilation, the build process uses a set of compiler options which
enable very strict error checking and asks the compiler to abort should it detect
any errors in the code. This is to ensure a maximum of code quality and
security.

Some (especially newer) compilers can be stricter than others when it comes
to error checking. While this is a good thing and the developers aim to address
all compiler warnings, it may lead the build process to abort on your system.

Should you notice error messages causing your build process to abort, do not
hesitate to contact the developers by creating a
[new issue report](https://github.com/greenbone/openvas/issues/new).
Don't forget to include the name and version of your compiler and distribution in your
message.


Setting up openvas
------------------

Setting up an openvas requires the following steps:

1. (optional) You may decide to change the default scanner preferences
   by setting them in the file `$prefix/etc/openvas.conf`. If that file does
   not exist (default), then the default settings are used. You can view
   them with `openvas -s`. The output of that command is a valid configuration
   file. The man page (`man openvas`) provides details about the available
   settings, among these opportunities to restrict access of scanner regarding
   scan targets and interfaces.

2. In order to run vulnerability scans, you will need a collection of Network
   Vulnerability Tests (NVTs) that can be run by openvas. Initially,
   your NVT collection will be empty. It is recommended that you synchronize
   with an NVT feed service before starting openvas for the first time.

   Simply execute the following command to retrieve the initial NVT collection:

       greenbone-nvt-sync

   This tool will use the Greenbone Security Feed in case a Greenbone
   subscription key is present. Else, the Community Feed will be used.

   Please note that you will need at least one of the following tools for a
   successful synchronization:

   * `rsync`
   * `wget`
   * `curl`

   NVT feeds are updated on a regular basis. Be sure to update your NVT collection
   regularly to detect the latest threats.

3. The scanner needs a running Redis server to temporarily store information
   gathered on the scanned hosts. Redis 3.2 and newer are supported.
   See `doc/redis_config.txt` to see how to set up and run a Redis server.

   The easiest and most reliable way to start redis under Ubuntu and Debian is
   to use systemd.

   ```bash
   sudo cp redis-openvas.conf /etc/redis/
   sudo chown redis:redis /etc/redis/redis-openvas.conf
   sudo echo "db_address = /run/redis-openvas/redis.sock" > /etc/openvas/openvas.conf
   sudo systemctl start redis-server@openvas.service
   ```

4. The scanner module does not run as a service as before any more. `gvmd`
   can act as a client and control the scanner through the `OSPD-OpenVAS`
   module. The actual user interfaces (for example GSA or GVM-Tools) will
   only interact with `gvmd` and/or ospd-openvas, not the scanner.
   You can launch openvas to upload the plugins in redis using the
   following command:

       openvas -u

   but ospd-openvas will do the update automatically each time a new feed
   is detected.

5. Please note that although you can run `openvas` as a user without elevated
   privileges, it is recommended that you start `openvas` as `root` since a
   number of Network Vulnerability Tests (NVTs) require root privileges to
   perform certain operations like packet forgery. If you run `openvas` as
   a user without permission to perform these operations, your scan results
   are likely to be incomplete.

   As `openvas` will be launched from an ospd-openvas process with sudo,
   the next configuration is required in the sudoers file:

       sudo visudo

   add this line to allow the user running `ospd-openvas`, to launch `openvas`
   with root permissions

       <user> ALL = NOPASSWD: <install prefix>/sbin/openvas

   If you set an install prefix, you have to update the path in the sudoers
   file too:

       Defaults        secure_path=<existing paths...>:<install prefix>/sbin


Logging Configuration
---------------------

If you encounter problems, by default the scanner writes logs to the file

    <install-prefix>/var/log/gvm/openvas.log

It may contain useful information.The exact location of this file may differ
depending on your distribution and installation method. Please have this file
ready when contacting the GVM developers via the Greenbone Community Portal
or submitting bug reports at <https://github.com/greenbone/openvas/issues> as
they may help to pinpoint the source of your issue.

Logging is configured entirely by the file

    <install-prefix>/etc/openvas/openvas_log.conf

The configuration is divided into domains like this one

    [sd   main]
    prepend=%t %p
    prepend_time_format=%Y-%m-%d %Hh%M.%S %Z
    file=/var/log/gvm/openvas.log
    level=128

The `level` field controls the amount of logging that is written.
The value of `level` can be

      4  Errors.
      8  Critical situation.
     16  Warnings.
     32  Messages.
     64  Information.
    128  Debug.  (Lots of output.)

Enabling any level includes all the levels above it. So enabling Information
will include Warnings, Critical situations and Errors.

To get absolutely all logging, set the level to 128 for all domains in the
configuration file.

Logging to `syslog` can be enabled in each domain like:

    [sd   main]
    prepend=%t %p
    prepend_time_format=%Y-%m-%d %Hh%M.%S %Z
    file=syslog
    syslog_facility=daemon
    level=128

Static code analysis with the Clang Static Analyzer
---------------------------------------------------

If you want to use the Clang Static Analyzer (https://clang-analyzer.llvm.org/)
to do a static code analysis, you can do so by prefixing the configuration and
build commands with `scan-build`:

    scan-build cmake ..
    scan-build make

The tool will provide a hint on how to launch a web browser with the results.

It is recommended to do this analysis in a separate, empty build directory and
to empty the build directory before `scan-build` call.
