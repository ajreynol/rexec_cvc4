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
- Multiple working copies of the cvc4 source in cloned git repositories,
- Multiple build configurations for each of these sources (e.g. prod/debug).

The client script works in cooperation with a server script `lexec_lnx`, which
can be installed on a remote machine `remotehost` via `rexec remotehost install-server`.
The basic goal is to use computing power on remote machines while developing
and testing cvc4.

# Setup (example)

Say we want:

* Two working copies of the cvc4 git repository: `exp` for experimental and `stb`
for stable.
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
In each of these directories, if desired, the developer may issue basic
git commands for switching branches, making commits, doing git merges and so on.
This workflow is intended to automatically adapt to the local git state (if the
developer manually does so), or otherwise provide ways of
managing git while being syncronized with remote machine(s).

The other four directories are the directories from which `rexec` should be
executed, which determines how to issue commands on remote machine(s).
For example, running:

`rexec remoteHost configure`

in working directory `localhome/build/stb/prod` signals a call to configure the
production build of the stable source on the remote machine `remoteHost`
(for details, see below).

Note that the local build directories under `build-cvc4/`
may optionally contain local build files
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
either be manual (`-s`) or automatically done through git.
The former may be preferred for performance and for keeeping a cleaner git log.

Additional command line arguments are passed (when applicable) to the remote
executable (e.g. to the corresponding `make regress` command that is run
remotely).

### Commands:

* `install-server`
Install a copy of the server script to the remote machine.
* `rinstall`
Remote install to local machine: builds a (static) binary on the remote machine,
and then copies this binary to `localhome/bin` on the local machine.
* `install|regress|ctest|units|clean`: 
Issues a command in the remote's build directory.
* `reset`
Delete the remote's build directory for the current (source, build config).
* `reset-all`
Delete the remote's entire build directory.
* `configure`
Runs configure.sh on remote for the current (source, build config).
* `info`
Print debug information on the local and remote source and binaries.
* `none`
Connect to the remote machine but do nothing.
* `checkout [branch name]`
Syntax sugar for `none -B [branch name]`.

###### Expert options:

* `exec [executable]`
Run executable from remote build directory
* `install-file [file]`
Install custom file from `localhome/bin/` to `remotehome/bin/` on the remote machine.

### Code syncronization options to rexec:

Note that the following synronization methods can be used by the developer
when desired. However, specifying a code syncronization method is optional.
If none is provided, then the server script implements a default policy
for determining the best action (see below).

* `-B [branch name]`
Switch to branch local and remote.
* `-c [commit message]`
Commit to local, rebase remote.
* `-s [syncronize type]`
Syncronize code local to remote. Use if you made code changes on the local machine that you don't want to commit to git.
Syncronization type is one of the following:
  - `src`: Syncronize the `src/` subdirectory.
  - `test`: Syncronize the `test/` subdirectory.
  - `dev`: Syncronize the `src/` and `test/` subdirectories.
  - `all`: Syncronize the entire local repository (not recommended).

The latter option is done manually by the developer when they know exactly
what portion of the source tree they modified.
Notice that using *any* syncronization option with `-s` has a special impact on
the policy of the server (see below). In particular, the server trusts that
all local changes are sent when any `-s` option is used.

Additionally, it is important to note that `-s` does *not* create new files
on the remote machine. Thus, adding a new file on the local machine must be
accompanied with a git commit and subsequent syncronization.

###### Expert options:

* `-b [branch name]`
Request a switch to branch on remote (not recommended to use this manually).
* `-r`
Request a rebase on remote (not recommended to use this manually).

# Examples:

* `rexec remoteHost regress -B testBranch`
Checkout branch `testBranch` on remote and local, run regressions on the remote `remoteHost`.
* `rexec remoteHost rinstall`
Install the current local branch on remote and copy the binary from the remote machine to the local machine. Notice this will throw an error if the local code has changed, based on the policy below.
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
| Yes           | Yes           | No              | Yes              | N/A               | force -r                      | Happens if local reverts changes after `-s`      |
| Yes           | Yes           | No              | No               | N/A               | (none)                        |                                                  |
| Yes           | No            | No              | N/A              | N/A               | force -r                      |                                                  |
| No            | No            | No              | N/A              | N/A               | force -b                      |                                                  |

Overall, this policy throws an error when the local and remote are
guaranteed to be out of sync, and a warning if it is unknown whether
local and remote are out of sync.

The above policy is implemented by having the client script communicate the local code state via auto-generated command
line arguments (`-lbranch`, `-lcommit`, `-lnomod`, `-lsync`) when calling the server script. Thus, in all
cases, the server script knows which case we are in. The above policy aims to do as little work
as possible while maintaining assurance that the code is syncronized. In five of the eight cases
above, the server does its best effort to syncronize by forcing a rebase `-r` or branch `-b`
but in two such cases, syncronization is known to be wrong (cases 3 and 4) and the server script aborts.
In one case, syncronization is unknown by the remote machine, and a warning is given instead (case 2).
