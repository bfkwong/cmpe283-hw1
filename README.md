# Group Members

1. Bryan Kwong

# Questions

## 1. For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched.

I did this homework by myself.

## 2. Describe in detail the steps you used to complete the assignment. Consider your reader to be someone skilled in software development but otherwise unfamiliar with the assignment. Good answers to this question will be recipes that someone can follow to reproduce your development steps.

### Step 1: Environment setup

#### Development Environment

The first thing I did was to make sure that I could run the sample code that was provided. I initiated a GCP VM and created a directory with the `Makefile` and the `cmpe283-1.c` file.

The next thing I did was to set up git on the VM so I could version track my progress. To do this I had to download git on my VM with `sudo apt-get install git`. Next, I have to generate an SSH key with `ssh-keygen -b 4096 -t rsa` and add the public SSH key in `id_rsa.pub` to my Github account to access the repo.

The next step that I did was to install all of the necessary libraries necessary for this homework assignment, which were:

1. `sudo apt install gcc make`
2. `sudo apt install linux-headers-$(uname -r)`

To help myself develop faster, I created a shell script called `start.sh` that would run all of the commands at once. I had to do `chmod 777 ./start.sh` to make the script executable. The content of `./start.sh` are below:

```
make
sudo insmod ./cmpe283-1.ko
sudo rmmod ./cmpe283-1.ko
sudo dmesg
make clean
```

#### Enabling Nested Virtualization on GCP VM

I followed this article to enable nested virtualization: https://cloud.google.com/compute/docs/instances/nested-virtualization/enabling#gcloud_1. The actionable steps that I took were the following: 

1. Shutdown my GCP VM instance named `cmpe283-hw1` because this cannot be done while running or suspended.
2. Downloaded gcloud CLI with their `./install.sh` script
3. Logged into gcloud CLI with `gcloud auth login`
4. Downloaded the VM instance config file with the following command. What differed from the the instructions was that I had to add the projectId via the `--project` tag.
```
gcloud compute instances export cmpe283-hw1 \ 
  --destination=./vmconfig.yaml \
  --zone=us-west1-b \
  --project=xxxxxxxx
```
5. Opened the `./vmconfig.yaml` file and added the following to the file.
```
advancedMachineFeatures:
  enableNestedVirtualization: true
```
6. Upload the `vmconfig.yaml` file to GCP. Note that the VM must be stopped for this to be successful
```
gcloud compute instances update-from-file cmpe283-hw1 \
  --source=./vmconfig.yaml \
  --most-disruptive-allowed-action=RESTART \
  --zone=us-west1-b \
  --project=xxxxxxxx
```
7. Start the VM and check to make sure that nested virtualization has been enabled. You can check by running `grep -cw vmx /proc/cpuinfo`. It is on if it returns a non-zero value.

### Step 2: Writing the Code

#### Printing all MSRs unconditionally

I started off by defining all the MSR indices at the top. These values are from the assignment PDF:

```
#define IA32_VMX_PINBASED_CTLS      0x481
#define IA32_VMX_PROCBASED_CTLS     0x482
#define IA32_VMX_PROCBASED_CTLS2    0x48B
#define IA32_VMX_EXIT_CTLS          0x483
#define IA32_VMX_ENTRY_CTLS         0x484
#define IA32_VMX_PROCBASED_CTLS3    0x492
```

Then I created the `capability_info` structs for the different controls. These values were copied these from the SDM directly. See the chapters below:

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
  // if the output is not 0, then continue into the if statemen
  if (hi & secondaryProcCtlMask) {

    // step 3: read secondary process control bit and print information.
    rdmsr(IA32_VMX_PROCBASED_CTLS2, lo, hi);
    pr_info("Secondary Process Controls MSR: 0x%llx\n",
      (uint64_t)(lo | (uint64_t)hi << 32));
    report_capability(procbasedctls2, 28, lo, hi);
  }
```

### Step 3: Testing

I ran my `./start.sh` script to test the output and got the following output.

```
[   59.566149] CMPE 283 Assignment 1 Module Start
[   59.571945] Pinbased Controls MSR: 0x3f00000016
[   59.577975]   External Interrupt Exiting: Can set=Yes, Can clear=Yes
[   59.584444]   NMI Exiting: Can set=Yes, Can clear=Yes
[   59.589613]   Virtual NMIs: Can set=Yes, Can clear=Yes
[   59.594860]   Activate VMX Preemption Timer: Can set=No, Can clear=Yes
[   59.601493]   Process Posted Interrupts: Can set=No, Can clear=Yes
[   59.609171] Primary Process Controls MSR: 0xf7b9fffe0401e172
[   59.614936]   Interrupt-window exiting: Can set=Yes, Can clear=Yes
[   59.622617]   Use TSC offsetting: Can set=Yes, Can clear=Yes
[   59.628391]   HLT exiting: Can set=Yes, Can clear=Yes
[   59.634929]   INVLPG exiting: Can set=Yes, Can clear=Yes
[   59.640445]   MWAIT exiting: Can set=Yes, Can clear=Yes
[   59.645778]   RDPMC exiting: Can set=Yes, Can clear=Yes
[   59.651113]   RDTSC exiting: Can set=Yes, Can clear=Yes
[   59.656456]   CR3-load exiting: Can set=Yes, Can clear=No
[   59.661967]   CR3-store exiting: Can set=Yes, Can clear=No
[   59.667571]   Activate tertiary controls: Can set=No, Can clear=Yes
[   59.673950]   CR8-load exiting: Can set=Yes, Can clear=Yes
[   59.679552]   CR8-store exiting: Can set=Yes, Can clear=Yes
[   59.685233]   Use TPR shadow: Can set=Yes, Can clear=Yes
[   59.690666]   NMI-window exiting: Can set=No, Can clear=Yes
[   59.696359]   MOV-DR exiting: Can set=Yes, Can clear=Yes
[   59.703161]   Unconditional I/O exiting: Can set=Yes, Can clear=Yes
[   59.709536]   Use I/O bitmaps: Can set=Yes, Can clear=Yes
[   59.715056]   Monitor trap flag: Can set=No, Can clear=Yes
[   59.720652]   Use MSR bitmaps: Can set=Yes, Can clear=Yes
[   59.727544]   MONITOR exiting: Can set=Yes, Can clear=Yes
[   59.733068]   PAUSE exiting: Can set=Yes, Can clear=Yes
[   59.739292]   Activate secondary controls: Can set=Yes, Can clear=Yes
[   59.747219] Secondary Process Controls MSR: 0x51ff00000000
[   59.752823]   Virtualize APIC accesses: Can set=Yes, Can clear=Yes
[   59.759119]   Enable EPT: Can set=Yes, Can clear=Yes
[   59.765569]   Descriptor-table exiting: Can set=Yes, Can clear=Yes
[   59.771863]   Enable RDTSCP: Can set=Yes, Can clear=Yes
[   59.777198]   Virtualize x2APIC mode: Can set=Yes, Can clear=Yes
[   59.783316]   Enable VPID: Can set=Yes, Can clear=Yes
[   59.788571]   WBINVD exiting: Can set=Yes, Can clear=Yes
[   59.793995]   Unrestricted guest: Can set=Yes, Can clear=Yes
[   59.799762]   APIC-register virtualization: Can set=Yes, Can clear=Yes
[   59.806406]   Virtual-interrupt delivery: Can set=No, Can clear=Yes
[   59.812781]   PAUSE-loop exiting: Can set=No, Can clear=Yes
[   59.818467]   RDRAND exiting: Can set=No, Can clear=Yes
[   59.823801]   Enable INVPCID: Can set=Yes, Can clear=Yes
[   59.829226]   Enable VM functions: Can set=No, Can clear=Yes
[   59.834996]   VMCS shadowing: Can set=Yes, Can clear=Yes
[   59.840424]   Enable ENCLS exiting: Can set=No, Can clear=Yes
[   59.846366]   RDSEED exiting: Can set=No, Can clear=Yes
[   59.851697]   Enable PML: Can set=No, Can clear=Yes
[   59.856681]   EPT-violation #VE: Can set=No, Can clear=Yes
[   59.863673]   Conceal VMX from PT: Can set=No, Can clear=Yes
[   59.869440]   Enable XSAVES/XRSTORS: Can set=No, Can clear=Yes
[   59.875393]   Mode-based execute control for EPT: Can set=No, Can clear=Yes
[   59.882463]   Sub-page write permissions for EPT: Can set=No, Can clear=Yes
[   59.889549]   Intel PT uses guest physical addresses: Can set=No, Can clear=Yes
[   59.896966]   Use TSC scaling: Can set=No, Can clear=Yes
[   59.903769]   Enable user wait and pause: Can set=No, Can clear=Yes
[   59.910144]   Enable PCONFIG: Can set=No, Can clear=Yes
[   59.915489]   Enable ENCLV exiting: Can set=No, Can clear=Yes
[   59.923686] Exit Controls MSR: 0x3fefff00036dff
[   59.928325]   Save debug controls: Can set=Yes, Can clear=No
[   59.935477]   Host address-space size: Can set=Yes, Can clear=Yes
[   59.941690]   Load IA32_PERF_GLOBAL_CTRL: Can set=No, Can clear=Yes
[   59.948068]   Acknowledge interrupt on exit: Can set=Yes, Can clear=Yes
[   59.956172]   Save IA32_PAT: Can set=Yes, Can clear=Yes
[   59.961502]   Load IA32_PAT: Can set=Yes, Can clear=Yes
[   59.966834]   Save IA32_EFER: Can set=Yes, Can clear=Yes
[   59.973647]   Load IA32_EFER: Can set=Yes, Can clear=Yes
[   59.979070]   Save VMX-preemption timer value: Can set=No, Can clear=Yes
[   59.985912]   Clear IA32_BNDCFGS: Can set=No, Can clear=Yes
[   59.991593]   Conceal VMX from Intel PT: Can set=No, Can clear=Yes
[   59.997888]   Clear IA32_RTIT_CTL: Can set=No, Can clear=Yes
[   60.003658]   Clear IA32_LBR_CTL: Can set=No, Can clear=Yes
[   60.010729]   Load CET state: Can set=No, Can clear=Yes
[   60.016070]   Load PKRS: Can set=No, Can clear=Yes
[   60.020974]   Save IA32_PERF_GLOBAL_CTL: Can set=No, Can clear=Yes
[   60.027276] Entry Controls MSR: 0xd3ff000011ff
[   60.031831]   Load debug controls: Can set=Yes, Can clear=No
[   60.038992]   IA-32e mode guest: Can set=Yes, Can clear=Yes
[   60.044685]   Entry to SMM: Can set=No, Can clear=Yes
[   60.049843]   Deactivate dual-monitor treatment: Can set=No, Can clear=Yes
[   60.058209]   Load IA32_PERF_GLOBAL_CTRL: Can set=No, Can clear=Yes
[   60.064599]   Load IA32_PAT: Can set=Yes, Can clear=Yes
[   60.069934]   Load IA32_EFER: Can set=Yes, Can clear=Yes
[   60.075375]   Load IA32_BNDCFGS: Can set=No, Can clear=Yes
[   60.080968]   Conceal VMX from PT: Can set=No, Can clear=Yes
[   60.088120]   Load IA32_RTIT_CTL: Can set=No, Can clear=Yes
[   60.093813]   Load CET state: Can set=No, Can clear=Yes
[   60.099147]   Load guest IA32_LBR_CTL: Can set=No, Can clear=Yes
[   60.106649]   Load PKRS: Can set=No, Can clear=Yes
[   60.119261] CMPE 283 Assignment 1 Module Exits
```
