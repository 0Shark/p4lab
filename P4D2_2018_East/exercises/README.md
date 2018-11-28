# P4 Tutorial

## Introduction

Welcome to the P4 Tutorial!

We've prepared a set of exercises to help you get started with P4
programming, organized into a set of modules:

1. Introduction and Language Basics (DVAD41)
* [Basic Forwarding](./basic)
* [Basic Tunneling](./basic_tunnel)
* [Explicit Congestion Notification](./ecn)

2. Load Balancing (DVAD42)
* [Load Balancing](./load_balance)
* [HULA](./hula)

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
