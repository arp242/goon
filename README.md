go-on runs Go on another machine, such as a QEMU VM or cloud instance; this is
useful for running tests or builds on different systems and architectures.

This requires zsh and ssh on the host machine, and qemu if you want to use qemu
VMs. On guests it requires starting a SSH server and `go`. I only tested the
host on my Linux machine, but I think it should run on BSD and macOS too (but I
didn't test it).

For example to run tests on Windows:

    [~cg/fsnotify]% go-on windows go test
    go-on: starting QEMU VM from windows-10.qcow2
    go-on: waiting for ssh to be available at localhost:9923 … Okay
    Version=10.0.19044
    PASS
    ok      github.com/fsnotify/fsnotify    5.843s

Or run a program on Windows:

    [~cg/fsnotify]% go-on windows go run ./cmd/fsnotify 'C:/Users/martin/fsnotify'
    go-on: windows already started; nothing to do
    Version=10.0.19044
    17:76:36.9001 watching; press ^C to exit

Any "go" command will work; let's try macOS:

    [~cg/fsnotify]% go-on macos go version
    go-on: starting QEMU VM from macos-12.qcow2
    go-on: waiting for ssh to be available at localhost:9924 … Okay
    macOS 12.5 21G72
    go version go1.18.4 darwin/amd64

Currently running machines:

    [~cg/fsnotify]% go-on machines
             freebsd    platform=freebsd    ssh=martin@localhost:9922
    started  macos      platform=macos      ssh=martin@localhost:9924
    started  windows    platform=windows    ssh=martin@localhost:9923

    [~cg/fsnotify]% go-on macos stop
    [~cg/fsnotify]% go-on windows stop

Setup
-----
Machines are defined in `machines/<name>`; everything is optional:

    # SSH connection details; it's recommended to use a non-privileged (>1024)
    # port, and to use a unique port or host for every machine so you can run
    # multiple at the same time, and won't have ssh complain about host key
    # verification.
    ssh_host=localhost                # Default: localhost
    ssh_port=9922                     # Default: 22
    ssh_user=martin                   # Default: $USER
    ssh_opts=(-i ~/.ssh/id_freebsd)   # Default: not set

    # Platform this machine is running; defaults to the machine name (i.e. the
    # file name)
    platform=freebsd

    # Start a new machine; see the scripts in machines/ for some QEMU examples.
    start-machine() {
        :
    }

See machines/ for some actual examples. All of these use QEMU at the moment, but
you run run `aws start` or whatever in `start-machine` too.

You may want to symlink the `go-on` command in your `$PATH`; for example:

    % ln -s ~/src/go-on/go-on ~/.local/bin

---

Setting up machines isn't done automatically; for QEMU [quickemu] might be
helpful here, especially for Windows and macOS.

Generally speaking you want to run sshd on port 22, setup
`~/.ssh/authorized_keys so you're not forever typing your password, and install
Go from the system's package repo or the installer from https://go.dev

Once everything works, creating a snapshot might be useful, so you have
something to fall back to later:

    % qemu-img snapshot -c init windows-10.qcow2

And then later you can revert to that with `-a init` if need be.

That said, there are some scripts and notes in the setup directory, but none of
this is very polished right now.

[quickemu]: https://github.com/quickemu-project/quickemu

Usage
-----
The syntax for is `go-on machine command`; the machine corresponds to a file in
the machines directory, and the command is one of:

	Go commands:
	    go         Start machine, sync directory, and run "go" (alias: test, build,
	               and run for "go test", "go build", and "go run").

	File commands:
	    get        Get a file.
	    put        Put a file.
	    sync       Sync the current directory.

	Management:
	    ssh        Run ssh (alias: shell, sh).
	    status     Get status of machine.
	    start      Start a machine (alias: boot).
	    stop       Stop a machine (alias: halt, shutdown).
	    machines   List all machines we know about (alias: list, ls).

There are no flags.

For example:

    % cd ~/mygoproject
    % go-on freebsd test -v

This will automatically start the machine if need be (or uses an already started
machine), syncs the directory, and runs "go test -v".

Stop the machine with `go-on freebsd stop` once you're done.

See `go-on help` for more detailed usage help.


FAQ
---
### Can't you just set GOOS/GOARCH?
For building as long as you don't need cgo: sure, that works brilliant. As soon
as you need cgo or very platform-specific things get complex or downright
impossible.

Running tests for a different GOOS is even more complex (impossible actually,
AFAIK).

### Why sync with ssh and not directory sharing?
It's network-transparent and fairly universal; I may want to run this on another
machine some day.

### Will it work for Python, C, Rust, or other languages?
With some modifications, sure. It just focuses on Go for now, as that's what I
need, and supporting just one thing with a bunch of assumptions built in keeps
things simple.

### Other solutions?
[Gomote](https://github.com/golang/go/wiki/Gomote) from the Go team, but you
can't really run that if you're not a member of the Go team. I looked a bit at
the code, and didn't seem easy to re-use with my own builders.
