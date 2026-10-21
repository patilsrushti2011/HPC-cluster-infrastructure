# xCAT Configuration

## Overview

xCAT is used to define and provision compute nodes in the HPC mini-cluster.

## Configuration Reference

- `node-definitions.example.txt` — documents selected node attributes, provisioning settings, operating system images, and post-provisioning scripts.

## Documented Configuration

- Architecture: x86_64
- Operating system: Rocky Linux 9.5
- Network boot method: xNBA
- Node profile: compute
- Node group: highmemory

## Post-Provisioning

The documented scripts include system logging, remote shell setup, file synchronization, HPC service startup, and Chrony configuration.

## Notes

The file is a reference derived from the project's node definition. It is not a replacement for the xCAT database or executable xCAT commands. Verify node attributes and operating system image names on the target server before making changes.
