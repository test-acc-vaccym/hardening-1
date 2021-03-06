# Hardening Playbook

## Abstract

An opinionated minimal-compromises guide to configuring a maximally secure
server for high stakes use cases where privacy and security are favored over
compatibility, cost, or effeciency.

This intends to be largely a showcase of the work of others and act as a
starting point for researching this space.

## Threat Profile

* Target protects:
  * Automated air/ground transportation
  * Nuclear weapons
  * Electric grid
  * Medical implants
  * Dive computers
  * Secrets that could end any entity
  * Access to unlimited financial gain
  * Human lives
* Attacker has
  * No ethics
  * unlimited funding
  * decades of patience
  * Knowledge deeper than yours of every component
  * 0-days of any currently known class
* Attacker can
  * compromise any single point in the supply chain
  * compromise any single system
  * compromise any single individual
* Attacker wants
  * Theft (cryptocurrency, bank accounts, stock tips, blackmail, databases)
  * Sabotage (to a company or country for any reason)
  * Chaos (May not be rational)

## Design

* Favor security and privacy over efficiency
* Every system:
  * is treated as a single purpose immutable appliance
  * replaced not updated
* Every component must be:
  * auditable by anyone
  * reproducible deterministically by anyone
  * audited by multiple reputable third parties.
  * fail on any unathorized physical tampering attempt
  * handle cryptographic operations in constant time
  * maintain secret keys physically separate from networks
  * have the bare minimum resources to complete its intended function

## Implementation

### Filesystem

Everything on unix is a file, and as such filesystem mount options and
permissions are one of the most effective ways to restrict what can or can't
be done in a given directory.

Everything should be either a read-only filesystem like quashfs or a tmpfs.
Never allow writes to root filesystem.

This is all managed via /etc/fstab

#### Overview

#### Recommendations

##### Mount options

###### Restrict /proc so users can only see their own processes
* Usage: ```proc /proc proc defaults,hidepid=2 0 0```

###### Disable suid binaries in /dev
* Usage: ```udev /dev devtmpfs defaults,nosuid,noexec,noatime 0 0```

###### Force mode 0666 in /dev/pts
* Usage: ```devpts /dev/pts devpts defaults,newinstance,ptmxmode=0666 0 0```

###### Use tmpfs for /dev/shm and restrict suid, exec, and dev
* Usage: ```tmpfs /dev/shm tmpfs defaults,nodev,nosuid,noexec 0 0```

###### Use tmpfs for /tmp and disable devices, suid binaries, and exec
* Usage: ```tmpfs /tmp tmpfs nodev,nosuid,noexec,size=2G 0 0```

###### Bind /var/tmp to /tmp and restrict suid, exec, and dev
* Usage: ```/tmp /var/tmp none rw,noexec,nosuid,nodev,bind 0 0```

##### Encryption

* TODO

### Toolchain

#### Overview



#### Recommendations

##### GCC/Binutils Options

###### Stack Canaries
* Usage: ```-fstack-protector-strong```
* Intention:
  * Plant random "canary" integers just before stack return pointers
  * Buffer overflows hijacking return pointer will normally modify canary
  * Ensure canary is still present before a routine uses a pointer on stack
* Resources:
  * https://outflux.net/blog/archives/2014/01/27/fstack-protector-strong/
  * https://en.wikipedia.org/wiki/Buffer_overflow_protection#Canaries

###### Position Independent Execution (PIE)
* Usage: ```-fPIE -pie```
* Intention:
  * Generated position independant code can only be linked into executables
  * When used in with ASLR, all program memory is allocated randomly together
  * Increase difficulty of using exploits that assume a specific memory layout
* Resources:
  * http://www.openbsd.org/papers/nycbsdcon08-pie/
  * https://mropert.github.io/2018/02/02/pic_pie_sanitizers/

###### Stack Clash Protection
* Usage: ```-fstack-clash-protection```
* Intention:
  * Mitigate attacks that rely on colliding neighboring memory regions
  * Defeats most historical stack clashing exploits
* Resources:
  * https://blog.qualys.com/securitylabs/2017/06/19/the-stack-clash
  * https://gcc.gnu.org/ml/gcc-patches/2017-07/msg01112.html

###### Data Execution Prevention (DEP)
* Usage: ```-Wl,-z,noexecstack -Wl,-z,noexecheap```
* Intention:
  * Buffer overflows tend to put code in programs stack and jump to it
  * If all writable addresses are non-executable, the attack is prevented
  * Don't mark memory as executable when it is not required
  * ELF headers are marked with PT_GNU_STACK and PT_GNU_HEAP
  * Set stacks/heaps to be executable only if segment flag calls for it
* Resources:
  * https://www.airs.com/blog/archives/518
  * https://linux.die.net/man/8/execstack

###### Source Fortification
* Usage: ```-DFORTIFY_SOURCE=2```
* Intention:
  * Many programs rely on functions that are not aware of buffer-length
  * Buffer overflow exploits often take advantage of these functions
  * Examples include strncpy, strcpy, memcpy, memset
  * Fail compilation if these are used in obviously unsafe way.
  * Compile with buffer-length aware checks for added run-time detection
  * Kill execution if buffer overflow check fires
* Resources:
  * https://idea.popcount.org/2013-08-15-fortify_source/

###### Run-time bounds checking for C++ strings/containers
* Usage: ```-Wp, -D_GLIBCXX_ASSERTIONS```
* Intention:
  * Turn on cheap range checks for C++ arrays, vectors, and strings
  * Add Null pointer checks when dereferencing smart pointers
* Resources:
  * https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_macros.html

###### No shared library text relocations
* Usage: ```-fpic -shared```
* Intention:
  *

###### Hardening Quality Control
* Usage: ```-plugin=annobin```
* Intention:
  *

###### Control flow integrity
* Usage: ```-mcet -fcf-protection```
* Intention:
  *

###### Reject potentially unsafe formt string args
* Usage: ```-Werror=format-security```
* Intention:
  *

###### Reject missing function prototypes
* Usage: ```-Werror=implicit-function-declaration```
* Intention:
  *

###### Detect and reject underlinking
* Usage: ```-Wl,-z,defs```
* Intention:
  *

###### Disable lazy binding
* Usage: ```-Wl,-z,now```
* Intention:
  *

###### RElocation Read-Only ELF Hardening
* Usage: ```-Wl,-z,relro```
* Intention:
  *

##### C/POSIX Standard Library Implementation

* TODO

#### Background

* https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html
* https://www.owasp.org/index.php/C-Based_Toolchain_Hardening#GCC.2FBinutils

### Kernel

#### Overview

While Linux is certianly not designed for out-of-the-box high security it is
the most portable for the widest range of use cases and has the largest number
of deployments so advice in this section will assume it.

If your application does not require a Linux kernel it is suggested the reader
carefully consider security-focused alternatives like OpenBSD, FreeBSD,
FreeRTOS, or seL4.

Some of these features don't ship with any published binary kernels for any
major distribution so it is assumed the reader will compile their own kernel
with a hardened toolchain following the advice in the Toolchain section of this
document.

#### Recommendations

##### Sysctl Options

###### Avoid kernel address exposures in /proc files (kallsyms, modules, etc).
* Usage: ```kernel.kptr_restrict = 1```
* Intention:
  *
* Notes:
* Resources:
  * Writeup:
[1]:

###### Avoid kernel memory address exposures via dmesg.
* Usage: ```kernel.dmesg_restrict = 1```
* Intention:
  *
* Notes:
* Resources:
  * Writeup:
[1]:

###### Block non-uid-0 profiling
* Usage: ```kernel.perf_event_paranoid = 3```
* Intention:
  *
* Notes: needs distro patch, otherwise this is the same as "= 2"
* Resources:
  * Writeup:
[1]:

###### Turn off kexec, even if it's built in.
* Usage: ```kernel.kexec_load_disabled = 1```
* Intention:
  *
* Resources:
  * Writeup:
[1]:

###### Avoid non-ancestor ptrace access to running processes and their credentials.
* Usage: ```kernel.yama.ptrace_scope = 1```
* Intention:
  * Avoid non-ancestor ptrace access to running processes and their credentials.
* Notes:
* Resources:
  * Writeup:
[1]:

###### Disable User Namespaces
* Usage: ```user.max_user_namespaces = 0```
* Intention:
  * removing large attack surface to unprivileged users.
* Notes:
* Resources:
  * Writeup:
[1]:

###### Disable unprivileged eBPF access.
* Usage: ```kernel.unprivileged_bpf_disabled = 1```
* Intention:
  *
* Notes:
* Resources:
  * Writeup:
[1]:

###### Turn on BPF JIT hardening, if the JIT is enabled.
* Usage: ```net.core.bpf_jit_harden = 2```
* Intention:
  *
* Notes:
* Resources:
  * Writeup:
[1]:

##### Config Flags

###### CONFIG_GCC_PLUGINS=y
* Intention:
  * Allow usage of static analysis plugins in GCC at kernel compile time
* Resources:
  * Writeup: [Kernel building with GCC Plugins][1]

[1]: https://lwn.net/Articles/691102/

###### CONFIG_GCC_PLUGIN_STACKLEAK=y
* Intention:
  * limit exfiltration of recycled stack memory
  * stack poisoning mitigation
  * runtime stack overflow detection
* Resources:
  * Talk: [STACKLEAK: A Long Way to the Linux Kernel Mainline - A. Popov][1]
  * Writeup: [How STACKLEAK improves Linux Security - A. Popov][2]
  * Patch: [#9778761](https://patchwork.kernel.org/patch/9778761/)

[1]: https://www.youtube.com/watch?v=5wIniiWSgUc
[2]: https://a13xp0p0v.github.io/2018/11/04/stackleak.html

###### CONFIG_GCC_PLUGIN_STRUCTLEAK=y
* Platforms: x86_64, arm64
* Intention:
  * Force all structure initialization before usage by other functions
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_GCC_PLUGIN_STRUCTLEAK_BYREF_ALL=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_GCC_PLUGIN_LATENT_ENTROPY=y
* Platforms: x86_64, arm64
* Intention:
  * Gather additional entropy at boot time as some systems have bad sources
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_GCC_PLUGIN_RANDSTRUCT=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_HARDENED_USERCOPY=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_CC_STACKPROTECTOR_STRONG=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_STRICT_KERNEL_RWX=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_RODATA=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEFAULT_MMAP_MIN_ADDR=65536
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_RANDOMIZE_BASE=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_RANDOMIZE_MEMORY=y
* Platforms: x86_64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_LEGACY_VSYSCALL_NONE=y
* Platforms: x86_64
* Intention:
  * Remove vsyscall entirely avoiding it as a fixed-position ROP target.
* Resources:
  * Writeup: [On vsyscalls and the vDSO][1]
  * Patch:

[1]: https://lwn.net/Articles/446528/

###### CONFIG_PAGE_TABLE_ISOLATION=y
* Platforms: x86_64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_IA32_EMULATION=n
* Platforms: x86_64
* Intention:
  * Disable 32 bit program emulation and all related attack classes.
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_X86_X32=n
* Platforms: x86_64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_MODIFY_LDT_SYSCALL=n
* Platforms: x86_64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_ARM64_SW_TTBR0_PAN=y
* Platforms: arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_UNMAP_KERNEL_AT_EL0=y
* Platforms: arm64
* Intention:
  * Kernel Page Table Isolation
  * Remove an entire class of cache timing side-channels.
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_BUG=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_STRICT_KERNEL_RWX=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_WX=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_STRICT_DEVMEM=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_IO_STRICT_DEVMEM=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SYN_COOKIES=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_CREDENTIALS=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_NOTIFIERS=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_LIST=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEBUG_SG=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_BUG_ON_DATA_CORRUPTION=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SCHED_STACK_END_CHECK=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SECCOMP=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SECCOMP_FILTER=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SECURITY=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SECURITY_YAMA=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_HARDENED_USERCOPY=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SLAB_FREELIST_RANDOM=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SLAB_FREELIST_HARDENED=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SLUB_DEBUG=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PAGE_POISONING=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PAGE_POISONING_NO_SANITY=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PAGE_POISONING_ZERO=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_VMAP_STACK=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_REFCOUNT_FULL=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_FORTIFY_SOURCE=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_ACPI_CUSTOM_METHOD=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_COMPAT_BRK=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_DEVKMEM=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PROC_KCORE=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_COMPAT_VDSO=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_KEXEC=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_HIBERNATION=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_BINFMT_MISC=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_LEGACY_PTYS=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_SECURITY_SELINUX_DISABLE=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PANIC_ON_OOPS=y
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_PANIC_TIMEOUT=-1
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### CONFIG_MODULES=n
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

##### Boot Options

###### slub_debug=P
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### page_poison=1
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### slab_nomerge
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

###### pti=on
* Platforms: x86_64, arm64
* Intention:
  *
* Resources:
  * Writeup:
  * Patch:

#### Background
* [Fedora Hardening Flags](https://fedoraproject.org/wiki/Changes/HardeningFlags28)
* [Android Kernel Hardening](https://source.android.com/devices/architecture/kernel/hardening)
* [ChromeOS Kernel Configs](https://chromium.googlesource.com/chromiumos/third_party/kernel/+/80b921861fdfebef21c2841ecc71d40b9d6b5550/chromeos/config/x86_64)
* [Debian Hardening](https://wiki.debian.org/Hardening)
* [Securing Debian Howto](https://www.debian.org/doc/manuals/securing-debian-howto/index.en.html#contents)
* [RedHat: Recommended GCC Compler Flags](https://developers.redhat.com/blog/2018/03/21/compiler-and-linker-flags-gcc/)
* [Debian Security Checklist](https://hardenedlinux.github.io/system-security/2015/06/09/debian-security-chklist.html)
* [System Down - HN discussion](https://news.ycombinator.com/item?id=18873530)
* [Why OpenBSD is Important To Me - HN Discussion](https://news.ycombinator.com/item?id=11660003)
* [Differences Between ASLR KASLR and KARL](http://www.daniloaz.com/en/differences-between-aslr-kaslr-and-karl/)
* [Linuxkit Security](https://github.com/linuxkit/linuxkit/blob/master/docs/security.md)

### Userspace

#### Recommendations

##### System Call Filtering
* TODO

##### Control Groups
* TODO

### Application

#### Recommendations

##### Code Signing
* TODO

##### Release Management
* TODO

##### Memory Management
* TODO
* Favor memory safe languages designed for security: Rust, Go, OCaml, Zig
* Consider a Hardened Memory allocator (hardened_malloc)

##### Third Party Dependencies
* TODO
* Signed reproducible builds must be possible
* Code must be signed with a well-known key of author and ideally reviewer(s)
* Consider reviews by any distribution channel maintainers
* Always assume force pushes and tag clobbers: pin hashes
* Assume upstreams will dissipear without warning: mirror everything yourself

#### Background
* TODO
* OpenBSD coding practices
