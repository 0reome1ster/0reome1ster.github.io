---
title: Windows Kernel Structures
date: 2024-10-28 10:00:00 -0500
categories: [windows, wke]
tags: [windows, wke, re]
author: oreo
dsecription: Windows Kernel Structures
---

## Introduction

This blog post serves as a reference guide/lookup table for me and those interested in Windows Kernel Exploitation (WKE) or Windows kernel malware development such as rootkits. It will cover various kernel structures that I find useful in my research and development projects.

## _EPROCESS

`_EPROCESS`, or `EPROCESS`, is an [opaque](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/eprocess) kernel structure that serves as the process object for a running process, both user- (such as your browser) and kernel-mode (such as your graphics driver). The `_EPROCESS` structure has a plethora of fields/members such as ptrs to `ActiveProcessLinks` and the process's `_TOKEN` structure.

---

## _TOKEN

`_TOKEN` is a kernel memory structure that describes an object's or process's security context - its privileges, logon id, session id, token type (e.g. primary or impersonation), and much more. A primary interest in this structure is its `_SEP_TOKEN_PRIVILEGES` field - see more [here](#_sep_token_privileges).

---

## _SEP_TOKEN_PRIVILEGES

---

## _KTHREAD

---