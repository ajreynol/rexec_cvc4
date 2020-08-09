# Remote development of cvc4

# Usage:

`rexec [remote host name] [remote command] [code sync options] [remote command options]`

See details on these arguments below.

# Basics

The goal of these scripts is enable a flexible development paradigm for cvc4
where remote machine(s) can be used while syncronizing code from a local
machine.

The script `rexec` is the main entry point that implements this development
paradigm. It handles cases where the developer is using:
- Multiple active copies of the cvc4 source in cloned git repositories,
- Multiple build configurations for each of these sources (e.g. prod/debug).

The client script works in cooperation with a server script `lexec_lnx`, which
can be installed on a remote machine `remotehost` via:

`rexec remotehost install-server`.

# Setup (example)

Say we want:

* Two working copies of the cvc4 git repository: `exp` for experimental and `stb`
for stable, which are actively being developed on the local machine in
parallel by the developer.
* Two build configurations: `debug` and `prod`, corresponding to debug
and production builds of cvc4.

On the local machine, the developer maintains the following working directories,
assuming a local home directory `localhome`:

* `localhome/cvc4-stb/`
* `localhome/build-cvc4/stb/prod/`
* `localhome/build-cvc4/stb/debug/`
* `localhome/cvc4-exp/`
* `localhome/build-cvc4/exp/prod/`
* `localhome/build-cvc4/exp/debug/`

The first and fourth contain working git clones of a (fork of) the cvc4 repo. 
The developer may issue git commands in these directories as desired, or
may otherwise use the `rexec` to manage certain git commands (see below).

The other four directories are working directories for issuing commands on
remote machine(s). For example, running:

`rexec remoteHost configure`

in working directory `localhome/build/stb/prod` runs a call to configure the
the stable source `localhome/cvc4-stb/` on the remote machine `remoteHost`
(for details, see below).

Note that the local build directories may optionally contain local build files
for the corresponding (source, build configuration) pairs. They can be
initialized by `configure_cvc4_local` running from the local source directories.
This is important in the case of loss of connection with remote machine(s), in
which case the developer can resort to building locally in the standard way
(e.g. `make regress`).

TODO: automatic setup of remote git.

# Usage

Assume a remote machine `remoteHost`, there are two main things to specifying
on the command line to `rexec remoteHost ...`:

* What command to execute on the remote machine.
* How do we want to syncronize the source code to the remote machine. This can
either be manual or through git. The former may be preferred for performance
and for keeeping a cleaner git log.

Additional command line arguments are passed (when applicable) to the remote
executable (e.g. to the corresponding `make regress` command that is run
remotely).

### Commands:

* `install-server`
Install a copy of the server script to the remote machine.
* `info`
Print debug information on the local and remote source and binaries.
* `rinstall`
Remote install to local machine: builds and copies a static binary of the current
(source, build config) to `localhome/bin` on the local machine.
* `reset`
Delete the remote's build directory for the current (source, build config).
* `reset-all`
Delete the remote's entire build directory.
* `configure`
Runs configure.sh on remote for the current (source, build config).
* `install|regress|ctest|units|clean`: 
Issues a command in the remote's build directory.
* `checkout [branch name]`
Syntax sugar for `info -b [branch name]`.

###### Expert options:
* `exec [executable]`
Run executable from remote build directory
* `install-file [file]`
Install custom file from local `bin/` to remote `bin/`.

### Code syncronization options to rexec:

* `-B [branch name]`
Switch to branch local and remote.
* `-b [branch name]`
Switch to branch on remote. Use this option if you changed branches manually on the local copy, independent of this script.
* `-c [commit message]`
Commit to local, rebase remote.
* `-r`
Rebase remote (not recommended to use this manually).
* `-s [syncronize type]`
Syncronize code local to remote. Use if you made code changes that you don't want to commit to git.

# Examples:

* `rexec remoteHost regress -B testBranch`
Checkout branch `testBranch` on remote and local, run regressions on remote.
* `rexec remoteHost rinstall -B testBranch`
Checkout branch `testBranch` on remote and local, install on remote and copy the binary from the remote machine to the local machine.
* `rexec remoteHost regress -s src`
Syncronize the local source to remote if the src/ directory was modified locally and run regressions.
* `rexec remoteHost regress CVC4_REGRESSION_ARGS=--tlimit=60000`
Run regressions with a 60 second timeout (determined by `CVC4_REGRESSION_ARGS` above). Code syncronization is automatically inferred when not provided based on the policy below.

# Policy for automatic code syncronization
 
Internally, the server script implements the following policy for code syncronization, where `N/A` indicates that the information is not relevant:

| Branch match? | Commit match? | Local Modified? | Remote Modified? | Used `-s`?        | Action:                       | Notes                                            |
|---------------|---------------|-----------------|------------------|-------------------|-------------------------------|--------------------------------------------------|
| Yes           | Yes           | Yes             | N/A              | Yes               | (none)                        |                                                  |
| Yes           | Yes           | Yes             | N/A              | No                | warning: use -s               | OK if local has no changes since last `-s`       |
| Yes           | No            | Yes             | N/A              | N/A               | force -r, error: use -s       |                                                  |
| No            | No            | Yes             | N/A              | N/A               | force -b, error: use -s       |                                                  |
| Yes           | Yes           | No              | Yes              | N/A               | force -r                      | Only happens if local reverts changes after `-s` |
| Yes           | Yes           | No              | No               | N/A               | (none)                        |                                                  |
| Yes           | No            | No              | N/A              | N/A               | force -r                      |                                                  |
| No            | No            | No              | N/A              | N/A               | force -b                      |                                                  |

This is implemented by having the client script communicate the local code state via command
line arguments (`-lbranch`, `-lcommit`, `-lnomod`, `-lsync`) to the server script. Thus, in all
cases, the server script knows which case we are in. The above policy aims to do as little work
as possible while maintaining assurance that the code is syncronized. In five of the eight cases
above, the server does its best effort to syncronize by forcing a rebase `r` or branch `b`
but in two such cases, syncronization is known to be wrong (cases 3 and 4) and the server script aborts.
In one case, syncronization is unknown by the remote machine, and a warning is given instead (case 2).
