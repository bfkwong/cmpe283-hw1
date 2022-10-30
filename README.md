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

To help myself make the development go faster, I created a shell script called `start.sh` that would run all of the commands at once. So I can test and develop quicker. I had to do `chmod 777 ./start.sh` to make the script executable.

./start.sh:

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
#define IA32_VMX_PINBASED_CTLS		0x481
#define IA32_VMX_PROCBASED_CTLS		0x482
#define IA32_VMX_PROCBASED_CTLS2  0x48B
#define IA32_VMX_EXIT_CTLS			0x483
#define IA32_VMX_ENTRY_CTLS			0x484
#define IA32_VMX_PROCBASED_CTLS3	0x492
```

Then I created the `capability_info` structs for the different controls. I had basically just copied and paste these from the SDM directly. These are where I got the control information from:

1. Exit capabilities: vol 3 - 24.7.1
2. Entry capabilities: vol 3 - 24.8.1
3. Primary process control capabilities: vol 3 - 24.6.2
4. Secondary process control capabilities: vol 3 - 24.6.2
5. Tertiery process control capabilities: vol 3 - 24.6.2

Initially I couldn't find `IA32_VMX_PROCBASED_CTLS3` but then I realized that I was looking at an old version of the SDM. I then had to double check the other controls to make sure that they were also up to date.

#### Conditional print of Secondary and Tertiery Process Controls

The next step was to check whether the secondary and tertiery process controls were enabled. This is at the 17th-indexed bit (tertiery) and the 31st-indexed bit (secondary) of the higher 32 bit of primary process control. The higher 32 bits tell us whether these features are enabled. If these bits were on, then we would print the secondary and tertiery capabilities.

This is done by doing a logical and on the higher 32 bit and a mask. To mask the 31st-indexed bit we would have to use `0x1` and to mask the 17th-indexed bit we would have to have `0x2000`.

1. Binary `00000000000000000000000000000001` -> Hex `0x1`
2. Binary `00000000000000001000000000000000` -> Hex `0x2000`

If the Primary Process Control MSR tertiary control was enabled, then the below would show

```
00110100100000001000001000010000
00000000000000001000000000000000
-------------------------------- &
00000000000000001000000000000000 -> 0x1
```

If the Primary Process Control MSR tertiary control was disabled, then the below would show

```
00110100100000000000001000010000
00000000000000001000000000000000
-------------------------------- &
00000000000000000000000000000000 -> 0x0
```

So when we do an and with the higher 32 control bits with `0x2000`, we get the output of `0x1` if the tertiery control bit was enabled and `0x0` if the tertiery control bit was disabled. We can use this to conditionally print the tertiery control information

### Step 3: Testing

I ran my `./start.sh` script to test the output. Everything printed out correctly and I went to get dinner.
