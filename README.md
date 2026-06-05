# VPS Hardening Guide

General-purpose security hardening baseline for freshly provisioned Linux VPS instances. Each guide targets a specific distribution and covers the same 12 sections as follows:

1. Operating System Baseline
2. User & Authentication
3. Secure Shell (SSH)
4. Network & Firewall
5. Package Integrity
6. Mandatory Access Control (MAC)
7. Logging & Audit
8. Kernel Hardening 
9. Filesystem Hardening
10. Performance Optimization
11. Intrusion Detection System
12. Maintenance Hygiene

**Scope:** Solo operator, newly provisioned VPS, no partition restructuring.  
**Format:** Hybrid checklist with rationale. Each section ends with a scriptable commands block.

## Available Guides

<table>
  <tbody>
    <tr>
      <td style="text-align:center; width: 120px; height: 150px; padding: 10px;">
        <img src="assets/AlmaLinux.svg" width="65"> <br/>
        <h2>Alma Linux</h2>
      </td>
      <td style="min-width: 500px">
        <b><a href="AlmaLinux/10/GUIDE.md">Guide for Alma Linux 10.1+ (Heliotrope Lion)</a></b>
        <br/> Version: AlmaLinux-10-Rev-0 | <a href="AlmaLinux-10/CHANGELOG.md">CHANGELOG</a>
        <hr/>
        <b><a href="AlmaLinux/9/GUIDE.md">Guide for Alma Linux 9.7+ (Moss Jungle Cat)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
        <hr/>
        <b><a href="AlmaLinux/8/GUIDE.md">Guide for Alma Linux 8.10+ (Cerulean Leopard)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; width: 120px; height: 150px; padding: 10px;">
        <img src="assets/Debian.svg" width="65"> <br/>
        <h2>Debian</h2>
      </td>
      <td style="min-width: 500px">
        <b><a href="Debian/13/GUIDE.md">Guide for Debian 13.x (Trixie | current "stable")</a></b>
        <br/> Version: Debian-13-Rev-0 | <a href="Debian/13/CHANGELOG.md">CHANGELOG</a>
        <hr/>
        <b><a href="Debian/12/GUIDE.md">Guide for Debian 12.x (Bookworm | current "oldstable")</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
        <hr/>
        <b><a href="Debian/11/GUIDE.md">Guide for Debian 11.x (Bullseye | current "oldoldstable" under LTS support)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
      </td>
    </tr>
    <tr>
      <td style="text-align:center; width: 120px; height: 150px; padding: 10px;">
        <img src="assets/Ubuntu.svg" width="65"> <br/>
        <h2>Ubuntu</h2>
      </td>
      <td style="min-width: 500px">
        <b><a href="Ubuntu/26.04/GUIDE.md">Guide for Ubuntu 26.04 LTS (Resolute Raccoon)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
        <hr/>
        <b><a href="Ubuntu/24.04/GUIDE.md">Guide for Ubuntu 24.04 LTS (Noble Numbat)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
        <hr/>
        <b><a href="Ubuntu/22.04/GUIDE.md">Guide for Ubuntu 22.04 LTS (Jammy Jellyfish)</a></b>
        <br/> <i>Work In-Progress (WIP)</i>
      </td>
    </tr>
  </tbody>
</table>
