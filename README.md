# Group Members

1. Bryan Kwong

# Questions

## 1. For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched.

I did this homework by myself.

## 2. Describe in detail the steps you used to complete the assignment. Consider your reader to be someone skilled in software development but otherwise unfamiliar with the assignment. Good answers to this question will be recipes that someone can follow to reproduce your development steps.

### Step 1: Development Environment setup

The first thing I did was to make sure that I could run the sample code that was provided. I initiated a GCP VM and created a directory with the `Makefile` and the `cmpe283-1.c` file.

The next thing I did was to set up git on the VM so I could version track my progress. To do this I had to download git on my VM with `sudo apt-get install git`. Next, I have to generate an SSH key with `ssh-keygen -b 4096 -t rsa ` and add this to my Github account to access the repo.

The next step that I did was to install all of the necessary libraries necessary for this homework assignment, they included:

1. `sudo apt install gcc make`
2. `sudo apt install linux-headers-$(uname -r)`

At this point I was a little confused because the `uname -r` responded with `5.10.0-18-cloud-amd64`. I wasn't sure if this meant that I had initiated the VM wrong. So I went back to check my VM config and it seemed like I did configure my VM correctly. I confirmed this by install `cpuid` and running it to see `vendor_id = "GenuineIntel"`.

To help myself make the development go faster, I created a shell script called `start.sh` that would run all of the commands at once. So I can test and develop quicker. I had to do `chmod 777 ./start.sh` to make the script executable. The content of `./start.sh` are below:

```
make
sudo insmod ./cmpe283-1.ko
sudo rmmod ./cmpe283-1.ko
sudo dmesg
make clean
```

### Step 2: Writing the Code

#### Making all of the MSRs everything print

I started off by defining all of the MSR indices at the top. I got these values from the assignment PDF

```
#define IA32_VMX_PINBASED_CTLS      0x481
#define IA32_VMX_PROCBASED_CTLS     0x482
#define IA32_VMX_PROCBASED_CTLS2    0x48B
#define IA32_VMX_EXIT_CTLS          0x483
#define IA32_VMX_ENTRY_CTLS         0x484
#define IA32_VMX_PROCBASED_CTLS3    0x492
```

Then I created the `capability_info` structs for the different controls. These values were copied these from the SDM directly. These are where I got the control information from:

1. `struct capability_info exitctls[16]`:
   Exit capabilities in vol 3 - 24.7.1
2. `struct capability_info entryctls[13]`:
   Entry capabilities in vol 3 - 24.8.1
3. `struct capability_info procbasedctls[22]`:
   Primary process control capabilities in vol 3 - 24.6.2
4. `struct capability_info procbasedctls2[28]`:
   Secondary process control capabilities in vol 3 - 24.6.2
5. `struct capability_info procbasedctls3[4]`:
   Tertiery process control capabilities in vol 3 - 24.6.2

Once these new structures have been created, we just need to enable it in `detect_vmx_features`. For example, for the exit controls, the following was created:

```
	/* Exit controls */
  // step 1: read the MSR from the defined address
	rdmsr(IA32_VMX_EXIT_CTLS, lo, hi);
  // step 2: print the MSR value
	pr_info("Exit Controls MSR: 0x%llx\n",
		(uint64_t)(lo | (uint64_t)hi << 32));
  // step 3: pass the MSR struct and the length into the provided `report_capability` function
	report_capability(exitctls, 16, lo, hi);
```

#### Conditional print of Secondary and Tertiery Process Controls MSRs

The next step was to check whether the secondary/tertiery process controls were enabled. Tertiery process control is at the 17th bit position and secondary process control is at the 31st bit position of the higher 32 bit of primary process control. We look at the higher 32 bits because that tell us whether these features are enabled; if these bits are on, then we would print the secondary/tertiery process controll capabilities.

This check is done by doing a logical `&` on the higher 32 bit of the primary process control MSR and a bit mask. To mask the 31st bit position for the secondary process controls, we would have to use `0b10000000000000000000000000000000` (`0x80000000` in hex). To mask the 17th bit position for the tertiery process controls, we would have to have `0b00000000000000100000000000000000` or (`0x20000` in hex). These values are defined in the code as follows:

```
uint32_t secondaryProcCtlMask = 0x80000000;
uint32_t tertiaryProcCtlMask = 0x20000;
```

If the secondary process control was enabled in the primary process control MSR, then the 31st bit position would be a 1. So the logical `&` of the primary process control MSR value with the secondaryProcCtlMask value would produce the output of `0x80000000`.

```
10110100100000001000001000010000 // primary process control MSR
10000000000000000000000000000000 // secondary process control bit mask
-------------------------------- &
10000000000000000000000000000000 -> 0x80000000
```

Similarly, if the secondary process control was disabled in the primary process control MSR, then the 31st bit position would be a 0. So the logical `&` of the primary process control MSR value with the secondaryProcCtlMask value would produce the output of `0x0`

```
00110100100000000000001000010000 // primary process control MSR
10000000000000000000000000000000 // secondary process control bit mask
-------------------------------- &
00000000000000000000000000000000 -> 0x0
```

Knowing that by taking the logical `&` of the primary process MSR value and the bit mask will generate an output of zero or non-zero based on whether the secondary/tertiery process controls are on, we can use this logic to conditionally print the secondary/tertiery process control information. We can do that with the following:

1. Read primary process MSR value.
2. Do a logical `&` with the primary process control MSR's higher 32 bit value and the bit mask. The output will be zero if secondary/tertiery process control bits are off and non-zero if the secondary/tertiery process control bits are on.
3. If the secondary/tertiery process control bits are on, then read secondary/tertiery process control MSR and print information

The code for the above described step is as follows

```
  // step 1: read primary process control MSR
	rdmsr(IA32_VMX_PROCBASED_CTLS,t lo, hi);
  // step 2: do logical between primary process control MSR's higher 32 bit and mask.
  //         if the output is not 0, then continue into the if statemen
	if (hi & secondaryProcCtlMask) {
    // step 3: read secondary process control bit and print information.
		rdmsr(IA32_VMX_PROCBASED_CTLS2, lo, hi);
		pr_info("Secondary Process Controls MSR: 0x%llx\n",
			(uint64_t)(lo | (uint64_t)hi << 32));
		report_capability(procbasedctls2, 28, lo, hi);
	}
```

### Step 3: Testing

I ran my `./start.sh` script to test the output. Everything printed out correctly and I went to get dinner.
