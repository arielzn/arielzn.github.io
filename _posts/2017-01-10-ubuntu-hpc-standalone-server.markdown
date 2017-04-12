---
layout: post
title:  "Ubuntu server setup as standalone HPC machine"
date:   2017-01-10 01:01:01 +0200
categories: hpc server
---


For a step-by-step guide on how to setup a Ubuntu Server 14.04 to be used for High Performance Computing see the detailed wiki of my Github project:

https://github.com/arielzn/modules_hpc_server/wiki


Despite the guide is intended for a machine exclusively used for HPC computing, the sections explaining the setup of Environment Modules (EM) package and how to build packages from source handled by EM, can be applied on a personal workstation also. This is very useful for someone developing Parallel applications, as that setup will allow to switch cleanly between different combinations of Compilers/Libraries/Parallel Interfaces, to verify the application behaves properly in different environments.


