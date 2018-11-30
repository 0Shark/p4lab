# Tutorials
**P4 language tutorials, adapted for the Data Plane Programming Course DVAD41**

We've adapted  a set of exercises to help you get started with P4 programming. They are a set of basic exercises,
some exercises focus on load balancing in data center networks and some on network monitoring.

The exercises have been originally developed for SIGCOMM 2017 and P4D2 2018 EAST. 

The exercises are in the folder /home/p4/tutorials/exercises. On this git, they are located 
[here](P4D2_2018_East/exercises/)
More exercises are available from 

 [Official P4 Github](https://github.com/p4lang/tutorials)
and 
 [Stanford CS344 class](https://github.com/CS344-Stanford-18)

## Obtaining required software

In order to complete the exercises, you will need to either build a
virtual machine or install several dependencies.

To build the virtual machine:
- Install [Vagrant](https://vagrantup.com) and [VirtualBox](https://virtualbox.org)
- `cd vm`
- `vagrant up`
- Log in with username `p4` and password `p4` and issue the command `sudo shutdown -r now`
- When the machine reboots, you should have a graphical desktop machine with the required
software pre-installed.

To install dependencies by hand, please reference the [vm](../vm) installation scripts.
They contain the dependencies, versions, and installation procedure.
You can run them directly on an Ubuntu 16.04 machine:
- `sudo ./root-bootstrap.sh`
- `sudo ./user-bootstrap.sh`




**Make sure that you edit [env.sh](env.sh) to point to your local copy of
  [bmv2](https://github.com/p4lang/behavioral-model) and
  [p4c-bm](https://github.com/p4lang/p4c-bm). You may want to follow the
  instructions
  [here](https://github.com/p4lang/tutorials/tree/master/SIGCOMM_2015#obtaining-required-software)
  to make sure that your environment is setup correctly. If you used vagrant up, it should work automatically and you do not need to edit the env.**
