# LDAP Configuration

## Overview

LDAP (Lightweight Directory Access Protocol) supports centralized directory-based identity lookup.

## Configuration Files

- `nslcd.conf.example` — documents the LDAP client URI and directory search base.
- `nsswitch.conf.example` — shows how the system checks local sources and LDAP for user, group, and related information.

## Settings

- LDAP URI: `ldap://master`
- Search base: `dc=hpc,dc=local`
- Client service user: `nslcd`
- Client service group: `ldap`

## Notes

These examples reflect the client-side settings documented for this project. The LDAP server configuration and authentication policies are not included. Verify the server hostname, search base, and security requirements before deployment.
