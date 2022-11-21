# Tutorials
**P4 language tutorials, adapted for the Data Plane Programming Course DVAD41**

We've adapted  a set of exercises to help you get started with P4 programming. They are a set of basic exercises,
some exercises focus on load balancing in data center networks and some on network monitoring.

The exercises have been originally developed for SIGCOMM 2017 and P4D2 2018 EAST. 

The exercises are in the folder /home/p4/vt19/P4lab/exercises. On this git, they are located 
[here](P4lab/exercises/).
More exercises are available from 

 [Official P4 Github](https://github.com/p4lang/tutorials)
and 
 [Stanford CS344 class](https://github.com/CS344-Stanford-18)

## Obtaining required software

In order to complete the exercises, you will need to either build a
virtual machine or install several dependencies.

To build the virtual machine:
- Install [Vagrant](https://vagrantup.com) and [VirtualBox](https://virtualbox.org)
- Clone the repository
- Before proceeding, ensure that your system has at least 12 Gbytes of free disk space, otherwise the installation can fail in unpredictable ways.
- `cd vm-ubuntu-20.04`
- `vagrant up` - The time for this step to complete depends upon your computer and Internet access speeds, but for example with a 2015 MacBook pro and 50 Mbps download speed, it took a little less than 20 minutes.  It requires a reliable Internet connection throughout the entire process.
- When the machine reboots, you should have a graphical desktop machine with the required software pre-installed.  There are two user accounts on the VM, `vagrant` (password `vagrant`) and `p4` (password `p4`).  The account `p4` is the one you are expected to use.

*Note*: Before running the `vagrant up` command, make sure you have enabled virtualization in your environment; otherwise you may get a "VT-x is disabled in the BIOS for both all CPU modes" error. Check [this](https://stackoverflow.com/questions/33304393/vt-x-is-disabled-in-the-bios-for-both-all-cpu-modes-verr-vmx-msr-all-vmx-disabl) for enabling it in virtualbox and/or BIOS for different system configurations.

You will need the script to execute to completion before you can see the `p4` login on your virtual machine's GUI. In some cases, the `vagrant up` command brings up only the default `vagrant` login with the password `vagrant`. Dependencies may or may not have been installed for you to proceed with running P4 programs. Please refer the [existing issues](https://github.com/p4lang/tutorials/issues) to help fix your problem or create a new one if your specific problem isn't addressed there.

To install dependencies by hand, please reference the [vm-ubuntu-20.04](./vm-ubuntu-20.04) installation scripts.
They contain the dependencies, versions, and installation procedure.
You should be able to run them directly on an Ubuntu 16.04 machine:
- `sudo ./root-bootstrap.sh`
- `sudo ./user-bootstrap.sh`




**Make sure that you edit [env.sh](env.sh) to point to your local copy of
  [bmv2](https://github.com/p4lang/behavioral-model) and
  [p4c-bm](https://github.com/p4lang/p4c-bm). You may want to follow the
  instructions
  [here](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#obtaining-required-software)
  to make sure that your environment is setup correctly. If you used vagrant up, it should work automatically and you do not need to edit the env.**

## P4 Cheat Sheet

A very nice summary of P4 commands and syntax is provided in the
 [P4 CheatSheet](P4lab/p4-cheat-sheet.pdf) 
 
## Presentation

The slides are available [online](http://bit.ly/p4d2-2018-spring) and
in the [P4 Tutorial](P4lab/P4_tutorial.pdf) .

## P4 Documentation

The documentation for P4_16 and P4Runtime is available [here](https://p4.org/specs/)

All excercises in this repository use the v1model architecture, the documentation for which is available at:
1. The BMv2 Simple Switch target document accessible [here](https://github.com/p4lang/behavioral-model/blob/master/docs/simple_switch.md) talks mainly about the v1model architecture.
2. The include file `v1model.p4` has extensive comments and can be accessed [here](https://github.com/p4lang/p4c/blob/master/p4include/v1model.p4).
# Older tutorials

Multiple live tutorial classes have been given using the example code
in this repository for hands-on exercises.  For example, there is one
each April or May at the P4 workshop at Stanford University in
California, and there have been several at networking conferences such
as ACM SIGCOMM.

Please [create an issue](https://github.com/p4lang/tutorials/issues)
for this tutorials repository if you know a public link for classroom
video recordings and/or pre-built VM images that currently do not have
such a link.


## ACM SIGCOMM August 2019 Tutorial on Programming the Network Data Plane

https://p4.org/events/2019-08-23-p4-tutorial/

The page linked above has a link to download a pre-built VM image used
for this class, as well as instructions to build one yourself from a
particular branch of this repository.


## P4 Developer Day, April 2019

https://p4.org/events/2019-04-30-p4-developer-day/

Both a beginner and advanced class were taught at this event.  The
page linked above contains instructions to download and install a
pre-built Linux VM that was used during the classes.


## P4 Developer Day, November 2017

* [YouTube
  videos](https://www.youtube.com/watch?v=3DJeqS_dl_o&list=PLf7HGRMAlJBzGC58GcYpimyIs7D0nuSoo)
  - This link plays the first welcome video of a series of 6 videos of
  tutorials given at this event.