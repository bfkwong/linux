# Homework 2 and 3 Questions

Group members: 

1. Bryan Kwong

## 1. What part of the assignment did you do?

I did the entire assignment by myself. 

## 2. Describe implementation steps in detail?

See changes at these 2 files: 

- `arch/x86/kvm/cpuid.c`
- `arch/x86/kvm/vmx/vmx.c`

### Setting up the Linux Kernel

First, we have to create a git fork of the Linux repo ([https://github.com/torvalds/linux](https://github.com/torvalds/linux)) and clone it onto our local machine. This is a really big repo so it might take a while to download. 

> ***Note:*** 2 issues I ran into while attempting to clone this repo 
> 1. **Git timeout**\
To resolve this error, increase the ssh timeout in the `~/.ssh/config` file. More details [here](https://confluence.atlassian.com/stashkb/git-ssh-push-timeout-on-large-changes-422020193.html).
> 2. **GCP VM Running out of memory**\
This issue was resolved by instantiating a "beefier" VM 

Once we have the Linux repo forked and cloned, we have to download all of the packages necessary to build and install the kernel. The following are the commands I ran

```
$ sudo apt-get install flex bison pkg-config libssl-dev bc
```

> ***Note:*** This may not be the complete list of packages needed as I was getting very frustrated at getting it to run that I stopped tracking mid way through.Long story short, just download whatever throws an error. 

Next we want to verify that the Linux kernel that we forked and cloned is working before we try modifying it. Before we can build the kernel, we have to make a config file. This can be done with `make oldconfig`

> ***Note:*** There will be a lot of options, so just default click through them all, this will likely throw an error at build but a quick search on Stack Overflow and an occasional 15 minute espresso break can help resolve them. 

We can test it with the following `make` commands 

```
$ make
$ make modules 
$ make modules_install
$ make install
```

This will likely take a really long time, but for me, it is November of 2022 and the World Cup is on so the wait wasn't that long. 

### Modifying `cpuid.c`

The first part of the code that we have to start with is in `cpuid.c`. In here, we will aim to do 2 things: 

1. Define global variables that will track these exit metrics and export them
2. Define special logic to handle the new EAX input

#### Define the Global Variable 

We can define the following global variables at the top level scope of the `cpuid.c` file. 

```
u32 total_exits = 0;
u64 total_exit_time = 0; 
u32 total_type_exits[75];
u64 total_type_exit_time[75];
```

We are using 32 bit variables for the total exit counts and 64 bit variables for total exit times. We define an array of length 75 because in the current SDM (volume 3 appendix C), they are numbered up to 69 exit types and I added 6 to make it a nice round 75. See discussion about using an array type to track the typed exits in the following sections. 

To export these variables, we use the `EXPORT_SYMBOL` function to export all of the global variables. 

```
EXPORT_SYMBOL(total_exits);
...
```

#### Defining Extra Handler 

We have to handle 4 new functions (as defined in the project assignment document). As the professor mentioned in the pre-recorded video, just create an if statement to check for `EAX` to determine how to handle the new functions. 

For handling EAX = 0x4FFFFFFE and EAX = 0x4FFFFFFF, you will have to use the value inside of ECX to index into the respective type arrays. So if CPUID was called with 0x4FFFFFFE and the ECX called for exit type 1. You will have to do the following to retrieve the total exit number for type 1. 

```
eax = total_type_exits[ecx]
```

The only other thing worth mentioning is that the following is how you get the higher 32 bit and the lower 32 bit from a `u64` integer.

```
u64 upper32_mask = 0xFFFFFFFF00000000;
u64 lower32_mask = 0x00000000FFFFFFFF;

ebx = (u32)((total_exit_time & upper32_mask) >> 32);
ecx = (u32)(total_exit_time & lower32_mask);
```


### Modifying `vmx.c`

The goal of the `vmx.c` modifications is just to populate the global variables that we defined in `cpuid.c`. 

The function we want to work on is `__vmx_handle_exit` but the trouble is that it is a rather complicated function with a lot of different exit locations. If we went to each of those exits and added modification code at each `return`, it would be a lot of code repetition and be difficult to refactor. 

The approach that I took is to rename the existing `__vmx_handle_exit` to `__handle_vmx_handle_exit` (clever name I know), and create a new `__vmx_handle_exit` with the same function signature. This way, it would be super easy to make modifications to the exit statistic tracking code. The additional benefit is that we would be including the invocation time of the handler as well. 

```
static int __handle_vmx_handle_exit(struct kvm_vcpu *vcpu, fastpath_t exit_fastpath) {
	...
}

static int __vmx_handle_exit(struct kvm_vcpu *vcpu, fastpath_t exit_fastpath) {
	extern u32 total_exits;
	extern u64 total_exit_time; 
	extern u32 total_type_exits[75];
	extern u64 total_type_exit_time[75];
	
	struct vcpu_vmx *vmx = to_vmx(vcpu);
	union vmx_exit_reason exit_reason = vmx->exit_reason;
	u64 handler_start_timestamp = rdtsc();
	u64 handler_end_timestamp;

	int vmx_exit_handler_handler_resp = __handle_vmx_handle_exit(vcpu, exit_fastpath);
	handler_end_timestamp = rdtsc() - handler_start_timestamp;

	total_exits += 1;
	total_exit_time += handler_end_timestamp;
	total_type_exits[exit_reason.basic] += 1;
	total_type_exit_time[exit_reason.basic] += handler_end_timestamp;

	return vmx_exit_handler_handler_resp;
}
```

#### Now technically this isn't right.

During the pre-recorded video, the professor mentioned that this technically isn't right because of atomic variables. It was less of a problem for `total_exits += 1` because the professor said smart compilers will use an atomic instruction for it. But I am not aware of a machine instruction that can read from array and increment. 

But then again, the professor also said we don't have to worry about atomics so there's that. 

### Putting it all together 

Remake all of your modules

```
$ make
$ make modules 
$ make modules_install
$ make install
```

Remove existing KVM modules and verify. The `lsmod | grep kvm` should not return anything,

```
$ rmmod kvm_intel
$ rmmod kvm 
$ lsmod | grep kvm 
```

Reload the KVM modules.

```
$ modprobe kvm
$ modprobe kvm_intel
```

That's all folks. If it doesn't work for you, this tutorial *"comes with ABSOLUTELY NO WARRANTY, to the extent permitted by applicable law."*

### Running the inner VM

In order to run the inner VM, you have to refer to assignment 1 to see how to enable nested virtualization on a GCP VM. If you have no idea what assignment 1 is, this GCP article can help ([https://cloud.google.com/compute/docs/instances/nested-virtualization/enabling](https://cloud.google.com/compute/docs/instances/nested-virtualization/enabling))

#### Choosing the inner VM OS 

As an OS nerd, this part is like walking through a candy shop. I looked for iso images that are in the `.qcow2` format, I think other formats work as well, but since [GCP advised](https://cloud.google.com/compute/docs/instances/nested-virtualization/creating-nested-vms) me to use `qemu-system-x86_64` to launch the VM, it made sense to use the `.qcow2` format. 

Unfortunately, at first it was really hard to find one an OS that worked. Ubuntu & Mint threw some error that I didn't know how to resolve, OpenBSD & RockyLinux would get stuck at a blinking cursor and never move forward. Ultimately, only 2 OS' ran for me: Debian 11 and CentOS 7. 

> ***Side note feel free to ignore:*** Still not over Red Hat killing CentOS.

#### Downloading the inner VM OS QCOW image

You can either upload it through the GCP VM SSH Console or use `wget`. In my experience, `wget` is way faster.

To download CentOS 7 with `wget` you can run the following. You can find the latest image [here](https://cloud.centos.org/centos/7/images/).

```
$ wget "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
```

#### Modifying the Nested VM root password

For some reason launching the nested VM will prompt you to enter a password, and because you didn't set one, you don't know the password. You will have to use the following command to reset the password in the image.

```
$ sudo apt-get install libguestfs-tools
$ virt-customize -a CentOS-7-x86_64-GenericCloud.qcow2 \
	--root-password password:PASSWORD
```

Now your login will be the following: 
- Login: `root`
- Password: `PASSWORD`

#### Starting the VM 

The GCP instruction is to install the following packages 

```
$ sudo apt update && sudo apt install qemu-kvm -y
```

Then run the following command 

```
$ sudo qemu-system-x86_64 -enable-kvm \
	-hda CentOS-7-x86_64-GenericCloud.qcow2 \
	-m 512 -nographic
```

The original GCP instructions is to use `-curses` but that didn't work for me, but digging through the man pages and stack overflow led me to use `-nographic` which did work. 

#### Inside of the Nested VM 

In the nested VM you will be able to run `cpuid`. Because I used CentOS as the nested VM, the command to install `cpuid` is 

```
$ sudo yum install cpuid
```

And then you can invoke `cpuid` with the following 

```
$ cpuid -l <EAX value> -s <ECX value>
```

## 3. Frequency of Exits 

### Frequency ~at Boot 

![](https://i.imgur.com/oSpcgL8.png)

### Frequency ~1 minute after boot

![](https://i.imgur.com/m1GQr70.png)

### Frequency ~2 minute after boot

![](https://i.imgur.com/lLi4wNg.png)

### Frequency ~3 minute after boot

![](https://i.imgur.com/7DVcYWp.png)

### Frequency ~4 minute after boot 

![](https://i.imgur.com/AMV1jgB.png)

### Frequency ~5 minute after boot

![](https://i.imgur.com/4wJ43qJ.png)

### Frequency ~6 minute after boot 

![](https://i.imgur.com/DhlWSZ9.png)

### Analysis

![](https://i.imgur.com/UgRIkkW.png)\


- The number of VM exits seems to keep increasing but does seem to slow down as time goes on
- For my VM there were 1,179,275 decimal number of exits after booting 
- Certain operations will trigger more VM exits than others

## 4. Exit types defined in SDM 

This is the printout of all of the CPUID exit types. This is just partial sample output of the exit types that had values. For full output, go to [https://drive.google.com/file/d/1yFToRCrdmN159PEACQuve9DPIIWerTOV/view?usp=sharing](https://drive.google.com/file/d/1yFToRCrdmN159PEACQuve9DPIIWerTOV/view?usp=sharing)

```
[root@localhost ~]# ./cpuid_test.sh 
CPU 0:
   0x4ffffffe 0x01: eax=0x00006f25 ebx=0x00000000 ecx=0x00000001 edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x07: eax=0x00006093 ebx=0x00000000 ecx=0x00000007 edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x0a: eax=0x00021fa5 ebx=0x00000000 ecx=0x0000000a edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x0c: eax=0x00006610 ebx=0x00000000 ecx=0x0000000c edx=0x00000000
CPU 0:
   0x4ffffffe 0x1c: eax=0x0000a891 ebx=0x00000000 ecx=0x0000001c edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x1d: eax=0x00000001 ebx=0x00000000 ecx=0x0000001d edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x1e: eax=0x000ebd0a ebx=0x00000000 ecx=0x0000001e edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x1f: eax=0x000008e4 ebx=0x00000000 ecx=0x0000001f edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x20: eax=0x0001ecd8 ebx=0x00000000 ecx=0x00000020 edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x30: eax=0x00000bbb ebx=0x00000000 ecx=0x00000030 edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x31: eax=0x000429ce ebx=0x00000000 ecx=0x00000031 edx=0xffff9c2d
CPU 0:
   0x4ffffffe 0x36: eax=0x00000006 ebx=0x00000000 ecx=0x00000036 edx=0xffff9c2d
```

The content of `cpuid_test.sh` is 

```
#!/bin/bash

for i in {1..75}
do 
  cpuid -l 0x4FFFFFFE -s "$i"
done
```

The 5 most frequent exits are: 

1. 0x1e/0d30 (I/O Instructions): 0xebd0a
2. 0x31/0d49 (EPT Misconfig): 0x429ce
3. 0xa/0d10 (CPUID): 0x21fa5
2. 0x20/0d32 (WRMSR): 0x1ecd8
3. 0x1c/0d28 (Control Register Access): 0xa891
