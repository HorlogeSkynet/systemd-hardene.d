# Security and hardening options for systemd service units

A common and reliable pattern in service unit files could be :

```
[Service]
MemoryDenyWriteExecute=yes

NoNewPrivileges=yes

PrivateBPF=yes
PrivateDevices=yes
PrivateIPC=yes
PrivateMounts=yes
PrivateTmp=yes

ProtectClock=yes
ProtectControlGroups=yes
ProtectHome=read-only
ProtectHostname=yes
ProtectKernelLogs=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
ProtectProc=noaccess
ProcSubset=pid
ProtectSystem=strict

RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes

LockPersonality=yes
SystemCallArchitectures=native
```

But there's so much more you can do. Here are some security-related options excerpting and linking to their corresponding documentation from systemd's manual pages :

## \[Unit\]

* [`ConditionPathIsEncrypted`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#ConditionPathIsEncrypted=) : Checks that the underlying file system's backing block device is encrypted using dm-crypt/LUKS.

* [`ConditionSecurity`](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#ConditionSecurity=) : May be used to check whether the given security technology is enabled on the system. Supported values are `selinux`, `apparmor`, `tomoyo`, `smack`, `ima`, `audit`, `uefi-secureboot`, `tpm2`, `cvm` and `measured-uki`. The test may be negated by prepending an exclamation mark.

## \[Service\]

### [Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)

* [`AmbientCapabilities`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#AmbientCapabilities=) : Controls which capabilities to include in the ambient capability set for the executed process. Takes a whitespace-separated list of capability names, e.g. `CAP_SYS_ADMIN`, `CAP_DAC_OVERRIDE`, `CAP_SYS_PTRACE`.

* [`CapabilityBoundingSet`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#CapabilityBoundingSet=) : Controls which capabilities to include in the capability bounding set for the executed process. See [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html) for details. Takes a whitespace-separated list of capability names, e.g. `CAP_SYS_ADMIN`, `CAP_DAC_OVERRIDE`, `CAP_SYS_PTRACE`.

### Devices

* [`DeviceAllow`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#DeviceAllow=) : Control access to specific device nodes by the executed processes. Takes two space-separated strings: a device node specifier followed by a combination of `r`, `w`, `m` to control reading, writing, or creation of the specific device nodes by the unit (*mknod*), respectively. When access to all physical devices should be disallowed, `PrivateDevices=` may be used instead (see below).

* [`DevicePolicy`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#DevicePolicy=auto%7Cclosed%7Cstrict) : Control the policy for allowing device access. It accepts :
	+ `strict` means to only allow types of access that are explicitly specified
	+ `closed` in addition, allows access to standard pseudo devices including */dev/null*, */dev/zero*, */dev/full*, */dev/random*, and */dev/urandom*
	+ `auto` (default) in addition, allows access to all devices if no explicit `DeviceAllow=` is present

* [`PrivateDevices`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateDevices=) : If set, sets up a new */dev/* mount for the executed processes and only adds API pseudo devices such as */dev/null*, */dev/zero* or */dev/random* (as well as the pseudo TTY subsystem) to it, but no physical devices such as */dev/sda*, system memory */dev/mem*, system ports */dev/port* and others. This is useful to turn off physical device access by the executed process.

### Filesystem

* [`ProtectHome`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectHome=) : Takes a boolean argument or the special values `read-only` or `tmpfs`. If true, the directories */home/*, */root*, and */run/user* are made inaccessible and empty for processes invoked by this unit. If set to `read-only`, the three directories are made read-only instead. If set to `tmpfs`, temporary file systems are mounted on the three directories in read-only mode. The value `tmpfs` is useful to hide home directories not relevant to the processes invoked by the unit, while still allowing necessary directories to be made visible when listed in `BindPaths=` or `BindReadOnlyPaths=`. Setting this to `yes` is mostly equivalent to setting the three directories in `InaccessiblePaths=`. Similarly, `read-only` is mostly equivalent to `ReadOnlyPaths=`, and `tmpfs` is mostly equivalent to `TemporaryFileSystem=` with `:ro`.

* [`ProtectSystem`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectSystem=) : Takes a boolean argument or the special values `full` or `strict`. If true, mounts the */usr/* and the boot loader directories (*/boot* and */efi*) read-only for processes invoked by this unit. If set to `full`, the */etc/* directory is mounted read-only, too. If set to `strict` the entire file system hierarchy is mounted read-only, except for the API file system subtrees */dev/*, */proc/* and */sys/* (protect these directories using `PrivateDevices=`, `ProtectKernelTunables=`, `ProtectControlGroups=`). This setting ensures that any modification of the vendor-supplied operating system (and optionally its configuration, and local mounts) is prohibited for the service. It is recommended to enable this setting for all long-running services, unless they are involved with system updates or need to modify the operating system in other ways. If this option is used, `ReadWritePaths=` may be used to exclude specific directories from being made read-only. This setting is implied if `DynamicUser=` is set.

* [`RestrictFileSystems`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictFileSystems=) : Restricts the set of filesystems processes of this unit can open files on. Takes a space-separated list of filesystem names, such as `ext4` or `tmpfs`. Any filesystem listed is made accessible to the unit's processes, access to filesystem types not listed is prohibited (allow-listing). If the first character of the list is "~", the effect is inverted: access to the filesystems listed is prohibited (deny-listing). If the empty string is assigned, access to filesystems is not restricted.

* [`UMask`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#UMask=) : Controls the file mode creation mask.

### Kernel

* [`ProtectKernelLogs`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectKernelLogs=) : If set, access to the kernel log ring buffer will be denied. It is recommended to turn this on for most services that do not need to read from or write to the kernel log ring buffer. Enabling this option removes `CAP_SYSLOG` from the capability bounding set for this unit, and installs a system call filter to block the [syslog(2)](https://man7.org/linux/man-pages/man2/syslog.2.html) system call. The kernel exposes its log buffer to userspace via */dev/kmsg* and */proc/kmsg*. If enabled, these are made inaccessible to all the processes in the unit.

* [`ProtectKernelModules`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectKernelModules=) : If set, explicit module loading will be denied. This allows to turn off module load and unload operations on modular kernels. It is recommended to turn this on for most services that do not need special file systems or extra kernel modules to work. Enabling this option removes `CAP_SYS_MODULE` from the capability bounding set for the unit, and installs a system call filter to block module system calls, also */usr/lib/modules* is made inaccessible. For this setting the same restrictions regarding mount propagation and privileges apply as for `ReadOnlyPaths=` and related calls. Note that limited automatic module loading due to user configuration or kernel mapping tables might still happen as side effect of requested user operations, both privileged and unprivileged.

* [`ProtectKernelTunables`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectKernelTunables=) : If set, kernel variables accessible through */proc/sys/*, */sys/*, */proc/sysrq-trigger*, */proc/latency_stats*, */proc/acpi*, */proc/timer_stats*, */proc/fs* and */proc/irq* will be made read-only and */proc/kallsyms* as well as */proc/kcore* will be inaccessible to all processes of the unit.

### Linux Security Modules

#### Mandatory Access Control

* [`AppArmorProfile`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#AppArmorProfile=) : Takes a profile name as argument. The process executed by the unit will switch to this profile when started. Profiles must already be loaded in the kernel, or the unit will fail. If prefixed by "-", all errors will be ignored. This setting has no effect if AppArmor is not enabled.

* [`SELinuxContext`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SELinuxContext=) : Set the SELinux security context of the executed process. If set, this will override the automated domain transition. However, the policy still needs to authorize the transition. This directive is ignored if SELinux is disabled.

* [`SmackProcessLabel`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SmackProcessLabel=) : Takes a `SMACK64` security label as argument. The process executed by the unit will be started under this label and SMACK will decide whether the process is allowed to run or not, based on it. The process will continue to run under the label specified here unless the executable has its own `SMACK64EXEC` label, in which case the process will transition to run under that label. When not specified, the label that systemd is running under is used. This directive is ignored if SMACK is disabled.

### [Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)

* [`RestrictNamespaces`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictNamespaces=) : Restricts access to Linux namespace functionality for the processes of this unit. Either takes a boolean argument, or a space-separated list of namespace type identifiers. If true, access to any kind of namespacing is prohibited. Otherwise, a space-separated list of namespace type identifiers must be specified, consisting of any combination of: *cgroup*, *ipc*, *net*, *mnt*, *pid*, *user*, *uts*, and *time*. By prepending the list with a single tilde character ("~") the effect may be inverted: only the listed namespace types will be made inaccessible, all unlisted ones are permitted (deny-listing). If the empty string is assigned, the default namespace restrictions are applied, which is equivalent to false.

#### Cgroup

* [`ProtectControlGroups`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectControlGroups=) : Takes a boolean argument or the special values `private` or `strict`. If true, the Linux Control Groups ([cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html)) hierarchies accessible through */sys/fs/cgroup/* will be made read-only to all processes of the unit. If set to `private`, the unit will run in a cgroup namespace with a private writable mount of */sys/fs/cgroup/*. If set to `strict`, the unit will run in a cgroup namespace with a private read-only mount of `/sys/fs/cgroup/`. Note `private` and `strict` are downgraded to false and true respectively unless the system is using the unified control group hierarchy and the kernel supports cgroup namespaces.

#### Clock

* [`ProtectClock`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectClock=) : If set, writes to the hardware clock or system clock will be denied. It is recommended to turn this on for most services that do not need modify the clock. Enabling this option removes `CAP_SYS_TIME` and `CAP_WAKE_ALARM` from the capability bounding set for this unit, installs a system call filter to block calls that can set the clock, and `DeviceAllow=char-rtc r` is implied. This ensures */dev/rtc\** are made read-only to the service. If this setting is on, but the unit doesn't have the `CAP_SYS_ADMIN` capability, `NoNewPrivileges=yes` is implied.

#### IPC

* [`PrivateIPC`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateIPC=) : Takes a boolean argument. If true, sets up a new IPC namespace for the executed processes. Each IPC namespace has its own set of System V IPC identifiers and its own POSIX message queue file system. This is useful to avoid name clash of IPC identifiers.

#### Mount

* [`PrivateMounts`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateMounts=) : If set, the processes of this unit will be run in their own private file system (mount) namespace with all mount propagation from the processes towards the host's main file system namespace turned off. This means any file system mount points established or removed by the unit's processes will be private to them and not be visible to the host.

* [`PrivateTmp`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateTmp=) : Takes a boolean argument, or `disconnected`. If enabled, a new file system namespace will be set up for the executed processes, and */tmp/* and */var/tmp/* directories inside it are not shared with processes outside of the namespace, plus all temporary files created by a service in these directories will be removed after the service is stopped. For this setting, the same restrictions regarding mount propagation and privileges apply as for `ReadOnlyPaths=` and related calls, see below. This setting is useful to secure access to temporary files of the process, but makes sharing between processes via */tmp/* or */var/tmp/* impossible. If set to `yes`, the backing storage of the private temporary directories will remain on the host's */tmp/* and */var/tmp/* directories. If `disconnected`, the directories will be backed by a completely new tmpfs instance, meaning that the storage is fully disconnected from the host namespace.

* [`ReadWritePaths`, `ReadOnlyPaths`, `InaccessiblePaths`, `ExecPaths`, `NoExecPaths`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ReadWritePaths=) : Sets up a new file system namespace for executed processes. These options may be used to limit access a process has to the file system. Each setting takes a space-separated list of paths relative to the host's root directory (i.e. the system running the service manager).

#### Network

* [`PrivateNetwork`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateNetwork=) : If set, sets up a new network namespace for the executed processes and configures only the loopback network device "lo" inside it. No other network devices will be available to the executed process. This is useful to turn off network access by the executed process.

#### PID

* [`ProtectProc`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectProc=) : Takes one of `noaccess`, `invisible`, `ptraceable` or `default` (which it defaults to). When set, this controls the `hidepid=` mount option of the "procfs" instance for the unit that controls which directories with process metainformation (*/proc/PID*) are visible and accessible: when set to `noaccess` the ability to access most of other users' process metadata in */proc/* is taken away for processes of the service. When set to `invisible` processes owned by other users are hidden from */proc/*. If `ptraceable` all processes that cannot be *ptrace()*'ed by a process are hidden to it. This option is implemented via file system namespacing, and thus cannot be used with services that shall be able to install mount points in the host file system hierarchy. Note that the root user is unaffected by this option, so to be effective it has to be used together with `User=` or `DynamicUser=yes`, and also without the `CAP_SYS_PTRACE` capability, which also allows a process to bypass this feature. It cannot be used for services that need to access metainformation about other users' processes.

* [`ProcSubset`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProcSubset=) : Takes one of `all` (the default) and `pid`. If `pid`, all files and directories not directly associated with process management and introspection are made invisible in the */proc/* file system configured for the unit's processes. This controls the `subset=` mount option of the "procfs" instance for the unit.

#### User

* [`PrivateUsers`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateUsers=) : If enabled, sets up a new user namespace for the executed processes and configures a user and group mapping. It accepts :
	+ `yes` or `self` : a minimal user and group mapping is configured that maps the `root` user and group as well as the unit's own user and group to themselves and everything else to the `nobody` user and group. This is useful to securely detach the user and group databases used by the unit from the rest of the system, and thus to create an effective sandbox environment. All files, directories, processes, IPC objects and other resources owned by users/groups not equaling `root` or the unit's own will stay visible from within the unit but appear owned by the `nobody` user and group.
	+ `identity` : user namespacing is set up with an identity mapping for the first 65536 UIDs/GIDs. Any UIDs/GIDs above 65536 will be mapped to the `nobody` user and group, respectively. While this does not provide UID/GID isolation, since all UIDs/GIDs are chosen identically it does provide process capability isolation, and hence is often a good choice if proper user namespacing with distinct UID maps is not appropriate.
	+ `full` : user namespacing is set up with an identity mapping for all UIDs/GIDs. In addition, for system services, it allows the unit to call *setgroups()* system calls (by setting */proc/pid/setgroups* to `allow`). Similar to `identity`, this does not provide UID/GID isolation, but it does provide process capability isolation. If this mode is enabled, all unit processes are run without privileges in the host user namespace (regardless of whether the unit's own user/group is `root` or not). Specifically this means that the process will have zero process capabilities on the host's user namespace, but full capabilities within the service's user namespace. Settings such as `CapabilityBoundingSet=` will affect only the latter, and there's no way to acquire additional capabilities in the host's user namespace.
	+ `managed` : a transient, dynamically allocated range of 65536 UIDs/GIDs is allocated for the unit, and a UID/GID mapping is assigned to the unit's process so the UID/GID 0 from inside the unit maps to the first UID/GID of the allocated mapping. Note that in this mode the UID/GID the service process will run as is different depending if looking from the host side (where it will be a high, dynamically assigned UID) or from inside the unit (where it will be 0). Also note that this mode will enable file system UID mapping for the file systems this service accesses, mapping the "foreign" UID range on disk to the selected dynamic UID range at runtime.

#### UTS (hostname)

* [`ProtectHostname`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#ProtectHostname=) : If set, sets up a new UTS namespace for the executed processes. In addition, changing hostname or domainname is prevented.

### [Seccomp](https://man7.org/linux/man-pages/man2/seccomp.2.html)

* [`LockPersonality`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#LockPersonality=) : If set, locks down the [personality(2)](https://man7.org/linux/man-pages/man2/personality.2.html) system call so that the kernel execution domain may not be changed from the default or the personality selected with `Personality=` directive. This may be useful to improve security, because odd personality emulations may be poorly tested and source of vulnerabilities.

* [`SystemCallArchitectures`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SystemCallArchitectures=) : Takes a space-separated list of architecture identifiers to include in the system call filter. If this setting is used, processes of this unit will only be permitted to call native system calls, and system calls of the specified architectures. The special identifier `native` implicitly maps to the native architecture of the system (or more precisely: to the architecture the system manager is compiled for).

* [`SystemCallFilter`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SystemCallFilter=) : Takes a space-separated list of system call names or system call groups. If this setting is used, system calls executed by the unit processes except for the listed ones will result in the system call being denied (allow-listing). If the first character of the list is "~", the effect is inverted: only the listed system calls will be denied (deny-listing). This option may be specified more than once, in which case the filter masks are merged. If the empty string is assigned, the filter is reset, all prior assignments will have no effect. The default action when a system call is denied is to terminate the processes with a `SIGSYS` signal. This can changed using `SystemCallErrorNumber=`.

### Miscellaneous

#### [BPF](https://man7.org/linux/man-pages/man2/bpf.2.html)

* [`PrivateBPF`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#PrivateBPF=) : If set, mount a private instance of the BPF filesystem on */sys/fs/bpf/*, effectively hiding the host bpffs which contains information about loaded programs and maps. Otherwise, if `ProtectKernelTunables=` is set, the instance from the host is inherited but mounted read-only.

#### IPC

* [`RemoveIPC`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RemoveIPC=) : If set, all System V and POSIX IPC objects owned by the user and group the processes of this unit are run as are removed when the unit is stopped. This setting only has an effect if at least one of `User=`, `Group=` and `DynamicUser=` are used. It has no effect on IPC objects owned by the root user. Specifically, this removes System V semaphores, as well as System V and POSIX shared memory segments and message queues. If multiple units use the same user or group the IPC objects are removed when the last of these units is stopped.

#### Memory

* [`MemoryDenyWriteExecute`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#MemoryDenyWriteExecute=) : If set, attempts to create memory mappings that are writable and executable at the same time, or to change existing memory mappings to become executable, or mapping shared memory segments as executable are prohibited. Specifically, a system call filter is added that rejects [mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html) system calls with both `PROT_EXEC` and `PROT_WRITE` set, [mprotect(2)](http://man7.org/linux/man-pages/man2/mprotect.2.html) or [pkey_mprotect(2)](http://man7.org/linux/man-pages/man2/pkey_mprotect.2.html) system calls with `PROT_EXEC` set and [shmat(2)](http://man7.org/linux/man-pages/man2/shmat.2.html) system calls with `SHM_EXEC` set. Note that this option is incompatible with programs and libraries that generate program code dynamically at runtime, including JIT execution engines, executable stacks, and code "trampoline" feature of various C compilers. This option improves service security, as it makes harder for software exploits to change running code dynamically. However, the protection can be circumvented, if the service can write to a filesystem, which is not mounted with noexec (such as */dev/shm*), or it can use *memfd_create()*. This can be prevented by making such file systems inaccessible to the service (e.g. `InaccessiblePaths=/dev/shm`) and installing further system call filters (`SystemCallFilter=~memfd_create`).

#### Networking

* [`IPAddressAllow`, `IPAddressDeny`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#IPAddressAllow=ADDRESS%5B/PREFIXLENGTH%5D%E2%80%A6) : Turn on network traffic filtering for IP packets sent and received over `AF_INET` and `AF_INET6` sockets. Both directives take a space separated list of IPv4 or IPv6 addresses, each optionally suffixed with an address prefix length in bits after a "/" character. If the suffix is omitted, the address is considered a host address, i.e. the filter covers the whole address (32 bits for IPv4, 128 bits for IPv6).

* [`RestrictAddressFamilies`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictAddressFamilies=) : Restricts the set of socket address families accessible to the processes of this unit. Takes `none`, or a space-separated list of address family names to allow-list, such as `AF_UNIX`, `AF_INET` or `AF_INET6`, see [address_families(7)](https://man7.org/linux/man-pages/man7/address_families.7.html) for all possible options. When `none` is specified, then all address families will be denied. When prefixed with "~" the listed address families will be applied as deny list, otherwise as allow list.

* [`RestrictNetworkInterfaces`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#RestrictNetworkInterfaces=) : Takes a list of space-separated network interface names. This option restricts the network interfaces that processes of this unit can use. By default, processes can only use the network interfaces listed (allow-list). If the first character of the rule is "~", the effect is inverted: the processes can only use network interfaces not listed (deny-list).

* [`SocketBindAllow`, `SocketBindDeny`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#SocketBindAllow=bind-rule) : Configures restrictions on the ability of unit processes to invoke [bind(2)](https://man7.org/linux/man-pages/man2/bind.2.html) on a socket. Both allow and deny rules to be defined that restrict which addresses a socket may be bound to.

#### Privileges

* [`NoNewPrivileges`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#NoNewPrivileges=) : If set, ensures that the service process and all its children can never gain new privileges through *execve()* (e.g. via setuid or setgid bits, or filesystem capabilities). This is the simplest and most effective way to ensure that a process and its children can never elevate privileges again. Defaults to false, but certain settings override this and ignore the value of this setting. This is the case when `SystemCallFilter=`, `SystemCallArchitectures=`, `RestrictAddressFamilies=`, `RestrictNamespaces=`, `PrivateDevices=`, `ProtectKernelTunables=`, `ProtectKernelModules=`, `MemoryDenyWriteExecute=`, `RestrictRealtime=`, `RestrictSUIDSGID=`, `DynamicUser=` or `LockPersonality=` are specified. Note that even if this setting is overridden by them, systemctl shows the original value of this setting. See also: [No New Privileges Flag](https://www.kernel.org/doc/html/latest/userspace-api/no_new_privs.html).

* [`RestrictSUIDSGID`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictSUIDSGID=) : If set, any attempts to set the set-user-ID (SUID) or set-group-ID (SGID) bits on files or directories will be denied (for details on these bits see [inode(7)](http://man7.org/linux/man-pages/man7/inode.7.html).

* [`SecureBits`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#SecureBits=) : Controls the secure bits set for the executed process. Takes a space-separated combination of options from the following list: `keep-caps`, `keep-caps-locked`, `no-setuid-fixup`, `no-setuid-fixup-locked`, `noroot`, and `noroot-locked`.

#### Scheduler

* [`RestrictRealtime`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#RestrictRealtime=) : If set, any attempts to enable realtime scheduling in a process of the unit are refused. This restricts access to realtime task scheduling policies such as `SCHED_FIFO`, `SCHED_RR` or `SCHED_DEADLINE`. See [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html) for details about these scheduling policies. Realtime scheduling policies may be used to monopolize CPU time for longer periods of time, and may hence be used to lock up or otherwise trigger Denial-of-Service situations on the system.
