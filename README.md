# Remote development of cvc4

# Usage:

`rexec [remote host name] [command] [code sync options] [remote command options]`

# Background:

The script `rexec` is used for managing multiple simultaneous builds of cvc4 via
remote machine(s). It handles:
- Multiple copies of the cvc4 source,
- Multiple build configurations for each of these sources (e.g. prod/debug).

As an example, say we want:

* Two working copies of the cvc4 git repository: `exp` for experimental and `stb`
for stable.
* Two build configurations: `debug` and `prod` for production.

On the remote machine, the user maintains the following directories:

> remoteHome/cvc4-stb/
remoteHome/cvc4-exp/

On the local machine, the user maintains the following directories:

> localHome/cvc4-stb/
localHome/cvc4-exp/
localHome/build/stb/prod/
localHome/build/stb/debug/
localHome/build/exp/prod/
localHome/build/exp/debug/

The first four contain working git clones of a (fork of) the cvc4 repo, which
will be syncronized pairwise between local and remote(s). The latter four
contain (optionally, in case of loss of connection to remote machines) contain
local build files for the corresponding (source, build configuration), which
can be initialized by `configure_cvc4_local` running from the local source
directories.

The user may issue remote calls from the local build directories, where the
current directory determines the remote (source, build configuration) copies.
For example, running:

> rexec myhostname configure

in working directory `localHome/build/stb/prod` runs the configure.sh for
using the source directory remoteHome/cvc4-stb with build configuration "prod".

The script allows for commands that test cvc4 (e.g. `regress`). It also supports
for creating and copying static binaries to the local machine (`rinstall`)
with a naming schema. In particular, a successful `rinstall` command from
`localHome/build/stb/debug/` will generate the static binary `debug-stb-cvc4`
on the local machine.

# Usage

The common use case of the script is to run for a fixed remote machine, call it
`remoteHost`. There are two main components of the remainder of the command
line:
* What command to execute on the remote machine.
* How do we want to syncronize the source code to the remote machine. This can
either be manual or through git. The former may be preferred for performance
and for keeeping a cleaner git log.

### Commands:

* `install-server`
Install a copy of the server script to the remote machine.
* `info`
Print debug information on the local and remote source and binaries.
* `rinstall`
Remote install to local. Builds and copies a static binary of the current
(source, build config) to `localHome/bin` on the local machine.
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

* `rexec remoteHost rinstall -B testBranch`
Checkout branch `testBranch` on remote and local, install the binary from the remote machine to the local machine.
* `rexec remoteHost regress -s src`
Syncronize the local source to remote if the src/ directory was modified locally and run regressions.
* `rexec remoteHost regress -B testBranch CVC4_REGRESSION_ARGS=--tlimit=60000`
Checkout branch `testBranch` on remote and local, run regressions with a 60 second timeout (determined by `CVC4_REGRESSION_ARGS` above).

# Policy for automatic code syncronization
 
Internally, the server script implements the following policy for code syncronization, where `N/A` indicates that the information is not relevant:

| Branch match? | Commit match? | Local Modified? | Remote Modified? | Was `-s` used? | Action:                 | Notes                                          |
|---------------|---------------|-----------------|------------------|-------------------|-------------------------|------------------------------------------------|
| Yes           | Yes           | Yes             | N/A              | Yes               | (none)                  |                                                |
| Yes           | Yes           | Yes             | N/A              | No                | `warning: use -s`         | OK if local has no changes since last `-s`       |
| Yes           | No            | Yes             | N/A              | N/A               | force -r, `error: use -s` |                                                |
| No            | No            | Yes             | N/A              | N/A               | force -b,` error: use -s` |                                                |
| Yes           | Yes           | No              | Yes              | N/A               | force -r                | Only happens if local reverts changes after `-s` |
| Yes           | Yes           | No              | No               | N/A               | (none)                  |                                                |
| Yes           | No            | No              | N/A              | N/A               | force -r                |                                                |
| No            | No            | No              | N/A              | N/A               | force -b                |                                                |

This is implemented by having the client script communicate the local code state via command
line arguments (`-lbranch`, `-lcommit`, `-lnomod`, `-lsync`) to the server script.
