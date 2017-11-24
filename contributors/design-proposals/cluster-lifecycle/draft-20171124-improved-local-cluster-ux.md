---
kep-number: draft-20171124
title: Improved Local Cluster User Experience
authors:
  - [**@errordeveloper**](https://github.com/errordeveloper)
  - [**@justincormack**](https://github.com/justincormack)
owning-sig: sig-cluster-lifecycle
participating-sigs:
  - sig-cli (TBD)
reviewers:
  - [**@dlorenc**](https://github.com/dlorenc)
  - [**@r2d4**](https://github.com/r2d4)
approvers:
  - TBD
editor: TBD
creation-date: 2017-11-24
last-updated: 2017-12-18
status: draft
see-also:
  - KEP-1
replaces:
  - local-cluster-ux.md
---

# Improved Local Cluster User Experience

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [User Stories [optional]](#user-stories-optional)
      * [Story 1](#story-1)
      * [Story 2](#story-2)
    * [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    * [Security Considerations](#security-considerations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Alternatives](#alternatives)

## Summary

This proposal formalises some of the ideas that community members had been prototyping and experimenting with in various modes in attempt to improve different
aspects of the user experience of Kubernetes on Linux, Windows or macOS workstations. It is in the best interest of the community to ensure that most desktop OS
users have decent experience and that we share tools and best practice techniques where we see fit.

## Motivation

At present many developers who need to run Kubernetes on their local workstation use minikube, with a few exceptions of Linux desktop users, who prefer to run
Kubernetes directly on their desktop OS without using a VM, and those who find different ways of running a local Kubernetes cluster.

Some users find that using a remote cluster is more practical for their use-cases, and that is perfectly understood, however this is not a dominant use-case
for multiple reasons which are beyond the scope of this proposal.

At present, minikube is very popular and works reasonably well for many user, however there are number of implementation details of minikube's dependencies that
on the surface result in a handful of minor user-experience issues that can be improved quite easily. Consequentially, the dependencies in question have better
and already mature substitutes. In essence, the goal of this proposal is to outline a plan to modernise minikube through adoption of tools and techniques that
are maintained by community members and thereby reduce some of the maintenance burden that exists at present due to early technical choices.

### Hypervisor Issues

There are a few popular hypervisors:

  - VirtualBox (macOS, Windows, Linux)
  - VMWare Fusion (macOS) and VMWare Workstation (Windows)
  - Parallels (macOS)
  - Hypervisor.framework and xhyve (macOS)
  - KVM and qemu (Linux)
  - HyperKit (macOS)
  - Hyper-V (Windows)

Minikube currently has some support for all of these options, however it's not always clear to the user which options to pick and how well each
will work. At present, minikube supports multiple options for each platform, and level of support varies. It's incredibly difficult for minikube
developers to test each of the options and setting CI is not feasible at present.

It's one of the goals of this proposal to pick best-known hypervisor solution for each platform and make it work best for the minikube users.

#### What minikube projects needs from a hypervisor?

There are a few key requirements for an ideal hypervisor:

  - native integrations with OS/hardware vendor
  - presence of an API that can be used to automate provisioning of VMs reliably
  - actively maintained
  - availability on popular versions of the given platform
  - close relationship between hypervisor maintainers and minikube project

VirtualBox is the only option that supports all of the popular platforms, but it doesn't match any of the requirements above.

#### Analysis

The macOS developer workstation market had been dominated by VMWare and VirtualBox for a long time, until Apple has released Hypervisor.framework recently
and open-source front-end app called xhyve has seen some adoption. Each of these options are problematic for minikube to recommend, e.g. VirtualBox is no longer
actively maintained and hasn't seen any new features in a long time (e.g. support for Hypervisor.framework [remains to be desired](https://www.virtualbox.org/ticket/14217)).

VMWare Fusion and Parallels require a commercial license, and thereby cannot be recommended as companions to minikube, which is free and open-source.

The xhyve project, on the other hand is open-source and supports the native Hypervisor.framework, but it has not been actively maintained by the original
author and has seen [very few contributions](https://github.com/mist64/xhyve/graphs/contributors). [HyperKit](https://github.com/moby/hyperkit) project by
Docker has started of as a fork of xhyve and is actively maintained by Moby Project community, it thereby fits the needs of minikube project very well.

Microsoft has added native hypervisor, Hyper-V, to Windows and ... ***(TODO)***

Hyper-V, unlike Hypervisor.framework on macOS, consists of back-end as well as front-end and thereby fits most of the needs of minikube project (except for
Windows 7 support, however Windows 7 is an old OS and supporting it would be a stretch goal for minikube project). Additionally, Microsoft team had been
engaging closely with Docker and Moby Project, hence there is close relationship within the community that can be relied upon.

Linux desktop users often use either KVM or VirtualBox, the reason VirtualBox is still popular with some users is that it has a UI and integrates with desktop
apps, which is not a desired feature for the minikube use-case. Those who wish to run desktop apps in a VM, should be able to do that separately, but running
desktop apps on Kubernetes is believed to be outside of the scope of minikube project. KVM and qemu front-end fit minikube needs perfectly.

Additionally, minikube developers ([**@dlorenc**](https://github.com/dlorenc) and [**@r2d4**](https://github.com/r2d4)) have already implemented experimental
support for [HyperKit](https://github.com/kubernetes/minikube/pull/1776) and moved [KVM driver](https://github.com/kubernetes/minikube/pull/1828) into the minikube
project in attempt to improve the user experience. Leveraging work that Docker team has done in HyperKit project, would be the best choice, as the primary use-case
of HyperKit (Docker for Mac) is nearly identical to the use-case of minikube project, and Docker team are part of the community, unlike e.g. VirtualBox team at
Oracle.

### Networking Issues with VirtualBox and VMWare

With both commonly used hypervisors VirtualBox and VMWare, the desktop host OS requires network configuration changes, unless guest VM requires no network access.

This is usually automated and doesn't impose a problem for a networking savvy user, however most of developer face significant challenges when they cannot
make their VM access local network and the Internet when their desktop OS is configured as a VPN client. It comes down to IP address range clashes between
VM network and the local network as well as the VPN, and routing table conflicts. In many enterprise use-cases disabling or reconfiguring VPN is not an option,
and it is not possible to auto-detect all addresses used on an enterprise network and resolve conflicts by adapting VM network configuration dynamically based
on the given local environment. It is also not feasible to entirely isolate the VM from the network.

[VPNKit](https://github.com/moby/vpnkit/#why-is-this-needed) (part of HyperKit) solves this issue by by reconstructing Ethernet traffic from the VM and
translating it into the relevant socket API calls on macOS or Windows. This allows the host application to generate traffic without requiring low-level Ethernet
bridging support. Additionally, VPNKit allows to expose services running on Kubernetes on `localhost`, without having to reference IP address of the minikube VM,
and supports transparent HTTP(S) proxies, which are common in enterprise networks.

### Hypervisor Abstraction Issue (`github.com/docker/libmachine`)

The minikube project has relied on libmachine since the very beginning, however this library has a number of issues, e.g. it requires functional networking
and uses SSH for bootstrapping live VMs, as opposed to the more robust immutable model, and it doesn't have a concrete definition of interfaces it depends
on (more on this in this [below](#bespoke-operating-system-issue). Moreover, it's no longer maintained beyond tracking latest version of Docker engine.

Additionally, libmachine supports non-desktop use-cases and thereby extends the scope of the problems it needs to solve properly (at least for minikube).
Namely, it has support for creating remove machines (not clusters) in multiple cloud providers. There are much more robust solutions for this problem and
no need for maintaining this kind of functionality in the minikube project.

One of the design flaws of libmachine is that when it provisions a VM it has to wait for SSH connection to become available, so it can execute steps via SSH.
This approach is prone to several failure modes, and is anti-pattern, in contrast to immutable approach as an alternative (where all configuration is already
built into the VM image and all it needs to do is start some processes on boot up).

It's a goal of this proposal to eliminate libmachine dependency.

### Bespoke Operating System Issue

The minikube project relies upon a bespoke OS, built with BitBake and only maintained by minikube project authors, this OS provides very little benefit to
the project and only imposes maintenance burden.

Is't one of the goals of this proposal to eliminate the need for maintaining the bespoke OS.

#### What minikube projects needs from an OS?

There are a few key requirements for an ideal hypervisor:

  - modern version of Linux
  - simple to build a minimal custom image with just-enough components and all the necessary components (no runtime software installation or re-configuration)
  - secure and reliable
  - simple and flexible (no hard dependencies on e.g. init system, package manager or absence of one)
  - (ideally) has first class support for containers
  - actively maintained
  - close relationship between OS maintainers and minikube project

#### Analysis

Using an off-the-shelf traditional Linux distribution would be one option, but it's very hard to trim Debian or RedHat system to fit the minimal image requirement.
Using Arch, Alpine or Slackware Linux as a base may seem plausible also, however the resulting image may end-up as bespoke as the current BitBake-based implementation.
There other options that lead to bespoke path, e.g. Gentoo, Yocto or Nix, but the goal is to stop maintaining a bespoke solution within minikube project.

On the other hand, CoreOS Container Linux and Fedora Atomic may seem like plausible options, and maintainers of both of these distributions have close relationship
with minikube project, however customising either of these distributions appears challenging to the author of this proposal, mostly due to build infrastructure
dependencies.
CoreOS Container Linux relies heavily on systemd, an nice kexec-based update system and doesn't have a package manager. Additionally, it's [very hard to customise][coreos],
and it would imply that minikube project would have maintain an Omaha backend for update system, and developers working on custom image would have to accustomed to
Gentoo/ChromeOS build infrastructure.
Fedora Atomic has a neat model based on RedHat package (RPM) manager and ships with systemd. Building custom Fedora Atomic image [is somewhat easier][atomic], however
it all revolves around RPM and tools associated with it, minikube project would also have to maintain an RPM repository.

[coreos]: https://coreos.com/os/docs/latest/sdk-modifying-coreos.html
[atomic]: http://www.projectatomic.io/docs/compose-your-own-tree

LinuxKit matches all of the requirements above, as it doesn't have a hard dependency on systemd. Although LinuxKit leverages Alpine Linux (and the APK package manager)
for most of its components, that is only by convention and any component can be replaced. Alternative implementation of LinuxKit component can be built as easily
as it is to build a Docker container image and thereby one can use any commodity tools under the hood. Additionally, one of the main use-cases of LinuxKit (Docker for Mac
and Windows) is very similar to the core use-case of minikube project. Additionally, LinuxKit defines what an image contains very explicitly through a packaging
system that features repeatable builds, so anyone consuming an image built with LinuxKit can find exactly what software is installed in the image, and there is
a concrete definition of the image for any external tools to rely on. Such external tools only need to pass key parameters via VM runtime metadata, and can safely
expect a VM to be fully bootstrapped once started, without resorting to SSH-based bootstrapping that depends on networking and usually has to assume how software
that is installed in the VM image works exactly, starting from basic assumptions about versions of utilities, through to permissions, implementation of the init
daemon and the order of stat up of certain system daemons.

### Single-binary Issue (`localkube`)

The minikube project include a bespoke distribution of Kubernetes – localkube. In localkube, all of the control plane components run in a single process
and copies of main functions of each of these components are maintained in the minikube project, which implies that minikube is different from any other
distribution of Kubernetes that one may run in production, and is hard to debug at the very least, but it also brings more maintenance burden. For every
new version of Kubernetes there have to be code updates in minikube. Besides that, it's hard for a Kubernetes contributor to test how their changes to
Kubernetes core may effect minikube users. Also, minikube is harder to customise (e.g. set a flag in one of the control plane components or use a different
version). Moreover, minikube users currently cannot select a network or storage add-on in the same way they would do it on a conventional/production-ready
distribution.

Additionally, [**@r2d4**](https://github.com/r2d4) has implemented [`kubeadm` bootstrapper](https://github.com/kubernetes/minikube/pull/1903) and actively
worked on improving it. LinuxKit already uses `kubeadm`, so that's another good reason to use LinuxKit.

### Other Issues

Some users find it inconvenient to run second VM aside from Docker for Mac or Windows VMs, because they wish to preserve resource usage, which often comes
down to energy. Some folks also find it inconvenient to have different Docker API endpoints for building containers. Both of these issue can be solved by
switching to Docker for Mac or Windows, hence these are outside of the scope of this proposal.

### Goals

- the outcome should be a new version of minikube that supports current CLI as closely as possible
- modernise minikube through adoption of tools and techniques that are maintained by community members
- reduce some of the maintenance burden that exists at present due to early technical choices
- eliminate the need for maintaining bespoke OS
- eliminate libmachine dependency
- pick best-known hypervisor solution for each platform and make it work best for the minikube users

### Non-Goals

- build more bespoke software
- build something that is significantly better then desktop platform it runs on

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories [optional]

<!--

Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of the system.
The goal here is to make this feel real for users without getting bogged down.

#### Story 1

#### Story 2

-->

### Implementation Details/Notes/Constraints

- might have to drop support from some implicit feature, e.g. HyperKit/VPNKit networking would work differently from VirtualBox or VMWare, but we maybe able to provide
  vmnet-based option as an alternative to VPNKit [TBD]
- would have to drop support for VirtualBox and thereby drop support for older versions of Windows (but this means better support for new versions of Windows) [TBD]
- Support for multiple CRI implementations (e.g. CRI-O and rkt), will need to back-ported to the new version of minikube, it's not very likely to happen immediately,
  however it maybe easier to solve with self-hosted CRI implementation [TBD]

### Security Considerations

Eliminating the bespoke OS will eliminate a number of risks, as the bespoke OS is not updated very often and doesn't consist of any security-minded aspects in its design.

LinuxKit, on the other hand, is more secure then current custom OS, as it's maintained by a wider community and is designed with security in mind. Some of the key aspects
include read-only filesystem, truly minimal software include in the base OS with all components, including Docker daemon and kubelet containerised.

***TODO***: Link to more information on LinuxKit security.

Additionally, supporting a smaller set of hypervisors eliminates a number of potential issues associated with hypervisor bugs. Preferred hypervisors are those that desktop
OS vendor ships, and thereby is expected to be as secure as the host OS.

## Graduation Criteria

***TBD***

<!--

How will we know that this has succeeded?
Gathering user feedback is crucial for building high quality experiences and SIGs have the important responsibility of setting milestones for stability and completeness.
Hopefully the content previously contained in [umbrella issues][] will be tracked in the `Graduation Criteria` section.

[umbrella issues]: https://github.com/kubernetes/kubernetes/issues/42752

-->

## Implementation History

***TBD***

<!--

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signalling SIG acceptance
- the `Proposal` section being merged signalling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

-->

## Alternatives

### Hypervisor Issues

We could recommend VirtualBox, it is not maintained but not actively developed, and no support for Hyper-V or Hypervisor.framework.
Also, VirtualBox is open source, but as it is an Oracle product and does appear to be open to external contributors, so improving it
doesn't seem feasible.

We could recommend VMWare or Parallels, but both of these require commercial license.

We could recommend xhyve on macOS, but xhyve is not actively maintained, and HyperKit is an improved and maintained fork of xhyve.

We could recommend Hyper-V Manager on Window, but it doesn't provide integrated user experience and would need to be automated.

### Bespoke Operating System Issue

...

### General

For the purpose of completeness of this proposal, it seems like a good idea to outline a few more alternative ideas of how one could approach the problem at hand,
but the authors of the proposal don't see any of these as an option.

#### Theoretically plausible, but not practically feasible

  - Make Kubernetes control plane easy to install and run natively on any desktop OS
    - AND only run a node in a VM or in user-space
    - OR make kubelet run natively on any desktop and with some kind of VM-based container runtime that is able to spin containers based on any OS
  - Assume developers should only work against remote Kubernetes clusters
    - Assume fast and ubiquitous Internet connectivity and no developer writes any code offline
    - Make remote cluster experience very rounded so that developers can use any of their desktop tools to write and debug code
    - Assume developers always commit code to git and there is always a CI pipeline that deploys their code to a cluster
    - Assume developers never need to provide direct access to any files from their local desktop filesystem to app code in a pod
    - Assume remote cluster can be easily and securely accessed from any desktop

#### Non-plausible

  - Assume developers using macOS desktop only work on apps for macOS and make sure Kubernetes works on macOS natively
  - Assume developers using Windows desktop only work on apps for Windows servers and make sure Kubernetes on Windows supports desktop OS
  - Make Kubernetes run natively on any desktop OS
  - Assume developers willing to use Kubernetes should use Linux on their desktop, perhaps their desktop OS should also run on Kubernetes
  - Everyone should just use Plan 9 and that should be the only OS Kubernetes above version 9 will support
