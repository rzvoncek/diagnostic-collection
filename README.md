This directory contains a set of scripts to generate a diagnostic tarball for DSE, DDAC &
open source Cassandra installations, similar (and partially compatible) to diagnostic
tarball generated by OpsCenter.

Generation of diagnostic tarball for DSE/Cassandra consists of 2 parts:
1. Collection of data on individual nodes;
2. Merging all data into single file.

These 2 steps could be separate because administrators may have different ways to access
nodes in the cluster, and transferring the data.  The `collect_diag.sh` performs data
collection from cluster as one command, executed on one of the nodes of the cluster, or
from Mac or Linux machine outside of the cluster.

## collect_diag.sh

This script copies the `collect_node_diag.sh` file to all nodes, executing it in parallel,
copying back collected data, and generate resulting tarball with diagnostic data using
[`generate_diag.sh`](#merging-all-diagnostics-into-single-tarball).

Usage:

```
./collect_diag.sh -t <type> [options] [install_root_path]
```

This script accepts the same arguments as [`collect_node_diag.sh`](#collecting-diagnostic-on-individual-nodes), 
with only `-t` as required parameter - all other are optional, but could be used to pass
SSH, nodetool, cqlsh, dsetool options.  For tarball installations it's required to pass
the path to the top-level directory of the DSE/DDAC/Cassandra installation (pass `-h` to
get a list of all options):

* `-c` - specifies options to pass to `cqlsh` (user name, password, etc.);
* `-d` - specifies the options for `dsetool` command (JMX user, password, etc.);
* `-f` - specifies the name where it should put the collected results (could be useful for
  some automation);
* `-m` - specifies the collection mode: `light`, `normal`, `extended` (default:
 `normal`). See [section below for information on what is collected](#what-is-collected):
 * `light` - collect only necessary information - logs, schema, nodetool, base system
 information;
 * `normal` - as previous, plus extended system information (iostat, vmstat, ...);
 * `extended` - all Cassandra logs, longer vmstat/iostat, etc. It could be significantly
   slower, and will generate bigger archive;
* `-n` - specifies a list of general options to pass to `nodetool` (JMX user, password, etc.);
* `-o` - specifies directory where to put where to put resulting file (default:
  automatically created);
* `-p` - specifies the PID of DSE/DDAC/Cassandra process.  Script tries to detect it from the output of
  `ps`, but this may not work reliably in case if you have several Cassandra processes on
  the same machine.  This PID is used to get information about limits set for process, etc.;
* `-r` - remove collected files after generation of resulting tarball
* `-s` - options to pass to SSH/SCP;
* `-t` (required) - specifies the type of installation: `dse`, `ddac`, `coss`;
* `-u` - specifies timeout for SSH in seconds (default: 600). You may need to increase it
  if you're using `extended` collection mode;
* `-v` - enables more verbose output by all scripts.

Please note that user name should be passed as `-o User=...` in `-s` option, as `scp` and
`ssh` are using different ways to pass user name.

Example of usage with list of hosts passed explicitly (file `mhosts`):

```sh
./collect_diag.sh -t dse -f mhosts -r -s \
  "-i ~/.ssh/private_key -o StrictHostKeyChecking=no -o User=automaton"
```

or for DDAC:

```sh
./collect_diag.sh -t ddac -f mhosts -r -s \
  "-i ~/.ssh/private_key -o StrictHostKeyChecking=no -o User=automaton" \
  /usr/local/lib/cassandra
```

if it's running on the machine with DSE/DDAC/C*, then it could be as simple as, as it will
use `nodetool status` to obtain a list of nodes:

```sh
./collect_diag.sh -t ddac -r /usr/local/lib/cassandra
```

## What is collected

The `collect_node_diag.sh` script collects following information:

* all DSE/DDAC/Cassandra configuration files - `cassandra.yaml`, `dse.yaml`, `/etc/defaults/dse`;
* all current log files in the `light` & `normal` collection modes, all log files,
  included rotated, in the `extended` mode;
* data from `nodetool` sub-commands, like, `tablestats`, `tpstats`, etc.;
* data from `dsetool` commands - at least `status` & `ring`;
* (in `extended` mode) executes for DSE the `nodetool sjk mxdump` to get all current values in JMX;
* database schema;
* schema and configuration for DSE Search cores;
* system information to help identify the problems caused by incorrect system settings
  (you may need to install some tools, like, `iostat`, `vmstat`, etc.):
  * information about CPUs, block devices, disks, memory, etc. (primarily from `/proc`
    filesystem);
  * information about operating system (name, version, etc.);
  * limits for user that runs DSE/DDAC/Cassandra;
  * output of `sysctl -a`, `dmesg`, etc.;
  * IO and VM statistics via `iostat` and `vmstat` (only in `normal` & `extended` modes).

**Important**: The `generate_diag.sh` script removes all sensitive information, such as,
passwords from configuration files.

## Collecting diagnostic on individual nodes

Collection of the data on individual nodes is performed by `collect_node_diag.sh` script
that executes different commands to collect data described in the previous section.
Script should be executed on every node of cluster, and if it's a tarball installation,
you need to pass one required parameter - full path to the root directory of
DSE/DDAC/Cassandra installation.  For package installation, location of the files will be
detected automatically, without specification of the root directory.  There are also
optional parameters, that could be provided if, for example, you have authentication
enabled for Cassandra or JMX, changed JMX port, etc. (pass `-h` to get list of options):

* `-c` - specifies options to pass to `cqlsh` (user name, password, etc.);
* `-d` - specifies the options for `dsetool` command (JMX user, password, etc.);
* `-f` - specifies the name where it should put the collected results (could be useful for
  some automation);
* `-m` - specifies the collection mode: `light`, `normal`, `extended` (default:
 `normal`). See [section below for information on what is collected](#what-is-collected):
 * `light` - collect only necessary information - logs, schema, nodetool, base system
 information;
 * `normal` - as previous, plus extended system information (iostat, vmstat, ...);
 * `extended` - all Cassandra logs, longer vmstat/iostat, etc. It could be significantly
   slower, and will generate bigger archive;
* `-n` - specifies a list of general options to pass to `nodetool` (JMX user, password, etc.);
* `-o` - specifies directory where to put where to put resulting file (default: `/var/tmp/`);
* `-p` - specifies the PID of DSE/DDAC/Cassandra process.  Script tries to detect it from
  the output of `ps`, but this may not work reliably in case if you have several Cassandra
  processes on the same machine.  This PID is used to get information about limits set for
  process, etc.;
* `-t` - specifies the type of installation: `dse`, `ddac`, `coss`  (default: `dse`);
* `-v` - enables more verbose output by all scripts.

After successful execution, script generates file with name
`/var/tmp/dse-diag-<IP_Address>.tar.gz`, like, `/var/tmp/dse-diag-10.200.179.237.tar.gz`,
or into specified by option `-f` (or if the `-w` flag was used, the file name prefix will
be `ddac-diag-...`).

## Merging all diagnostics into single tarball

After the `collect_node_diag.sh` script was executed on every machine of the cluster,
generated files should be collected into single directory on one machine to be merged
using the `generate_diag.sh` script.  This script accepts single parameter - path to
directory with collected files (it could be either relative, or absolute).  There are also
optional parameters:

* `-f` - specifies path to file where data should be put;
* `-o` - specifies directory where to put where to put resulting file (default: `/var/tmp/`);
* `-p` - specifies the custom pattern for file names if `collect_node_diag.sh` was called
  with `-f` parameter - otherwise this script may not find the data;
* `-r` - remove individual diagnostic files after processing;
* `-t` - specifies the type of installation: `dse`, `ddac`, `coss` (default: `dse`).

This script performs following:

* Creates a temporary directory in the `/var/tmp` or specified by `-o` flag;
* Unpacks each of collected files;
* Removes sensitive information, such as, passwords from configuration files;
* Packs everything together into single file that has name
  `<cluster_name>-diagnostics.tar.gz`, for example, `dsetest-diagnostics.tar.gz`, or into
  specified by `-f` option.

This file could be then sent for analysis to DataStax support, or analyzed by tools, like, [sperf](https://github.com/DataStax-Toolkit/sperf).

