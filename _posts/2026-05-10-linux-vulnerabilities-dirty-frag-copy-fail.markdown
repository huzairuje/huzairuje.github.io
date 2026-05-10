---
layout: post
title: "Critical Linux Vulnerabilities Disclosed: Dirty Frag and Copy Fail"
date: 2026-05-10 18:30:00 +0700
categories: linux security vulnerabilities
tags: [linux, kernel, security, dirty-frag, copy-fail, cve-2026-31431]
description: "A summary of two newly discovered Linux kernel vulnerabilities, Dirty Frag and Copy Fail (CVE-2026-31431), their impact, status, and mitigations."
---

This month has seen the disclosure of two significant vulnerabilities affecting the Linux kernel's page cache processing. Both flaws can lead to unauthorized root access, making them critical issues for system administrators to address.

## Dirty Frag

Discovered by researchers through manual analysis of the Linux kernel's page cache, this zero-day flaw tricks the kernel into writing restricted file data into a vulnerable memory area.

**The Impact:** Allows any local user to obtain a root shell with high success rates across most distributions.

**The Status:** Because of a broken disclosure embargo, the exploit is currently public. There are no assigned CVEs or official patches available at this time.

**Exploit / PoC:** [https://github.com/V4bel/dirtyfrag/tree/master](https://github.com/V4bel/dirtyfrag/tree/master)

**The Mitigation:** You can temporarily mitigate the bug by running commands to prevent `ESP4`, `ESP6`, and `RXRPC` modules from loading, and clearing the page cache.

## Copy Fail (CVE-2026-31431)

Just one week prior to Dirty Frag, security researchers disclosed the Copy Fail bug, a high-severity (CVSS 7.8) vulnerability that also affects how the kernel's page cache processes data.

**The Impact:** Unprivileged users can overwrite protected files in cache memory, leading to unauthorized root access.

**The Status:** CISA quickly added this 9-year-old bug to its Known Exploited Vulnerabilities catalog.

**Exploit / PoC:** [Find Copy Fail on GitHub](https://github.com/search?q=CVE-2026-31431)

**The Fix:** Major distributions like AlmaLinux and Rocky Linux have released updated kernel versions to fix the vulnerability. Users are strongly advised to run `sudo apt update && sudo apt upgrade` (on Debian/Ubuntu systems) or `sudo dnf update` (on RHEL-based systems).
