# Introduction

## Objective

This specification defines the security considerations in jiocloud design.

## Scope

### In scope
* Operating system security 

 * Root user login must be disabled on all production systems.
    Root user login should be disallowed in servers which are in production. In case of a hardware or operating system level issue, the system must be taken out of production. Once the server taken out of production, one can reboot the machine in single user mode and access the system. 

 * Single user mode must be password protected

 * Sudo users must have password - disallow NOPASSWD in sudoers

 * Direct console login must be disabled - no users must be able to login to direct console in production system.

 * Limit or eliminate the need of remote login to production systems

   * developing tools for operations 

   * Considering tools like hubot, which has integration with irc, and other chat application, so in effect operatiion activities can be done through private irc

 * Log consolidation

 * Access Audit


* Network security 

 * SSL encryption right upto the app servers

 * Network Scanning

 * NIDS/IPS

* Things to be considered as secrets while pushing the code and data to public
 
 * User list
 * Keys and passwords
 * Internal network and system details
 * SSL keys


