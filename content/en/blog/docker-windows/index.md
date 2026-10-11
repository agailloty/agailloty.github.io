---
title: "Installing Docker on Windows 10/11"
description: "Learn how to install Docker on a Windows 10/11 PC"
date: 2023-08-08T00:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [docker, devops, windows, Csharp, programming]
---

# Introduction

Docker is an open-source platform that simplifies deploying and managing applications in containers. Containers are isolated, portable environments that let developers package an application with its dependencies for consistent execution across different systems. This guide will help you install Docker on a Windows 10 PC, with detailed steps for a successful installation.

# Step 1: Checking the prerequisites

Before installing Docker, make sure your system meets the minimum requirements.
There are two ways to install Docker on Windows:

- Through WSL
- Through Hyper-V

The requirements depend on which option you choose.
Note that only the WSL installation is compatible with Windows 10/11 Home, Pro, Enterprise or Education editions.

## Prerequisites for installing Docker via WSL

* WSL version 1.1.3.0 or higher.
* Windows 10 64-bit: Home or Pro 21H2 (build >= 19044), or Enterprise or Education 21H2 (build >= 19044).
* Windows 11 64-bit: Home or Pro version 21H2 or higher, or Enterprise or Education version 21H2 or higher.
* The WSL 2 feature enabled on Windows.

## Prerequisites for installing Docker via Hyper-V

Windows 10 64-bit: Pro 21H2 (build >= 19044) or Enterprise or Education 21H2 (build >= 19044).
Windows 11 64-bit: Pro 21H2 (build >= 19044) or Enterprise or Education 21H2 (build >= 19044).
You will also need to enable the Hyper-V feature on your computer.

The Hyper-V prerequisites are given here for information only; in this article we will install Docker via WSL2. Note that installing Docker via Hyper-V gives you the ability to use Windows containers.

# Installing Docker

Docker is a tool that can package an application and its dependencies into an isolated container, which can then run on any server.

The problem is that we want to install Docker on Windows, and the Linux kernel is different from Windows and not compatible. So we need a way to run a Linux operating system on Windows: that is exactly what WSL allows.
Windows Subsystem for Linux (WSL) is a compatibility layer for running Linux binary executables natively on Windows 10, Windows 11 and Windows Server 2019. WSL 2 was released in May 2019.

## Enabling WSL 2

You first need to enable WSL2 on your computer. Make sure virtualization is allowed in your BIOS.

Alternatively, you can enable this feature with PowerShell.
If you are on a Windows 10 build >= 2004 or on Windows 11, the following command installs everything WSL needs to work.

```pwsh
wsl --install
```

## Download Docker Desktop

Docker Desktop is the fastest way to install Docker on Windows. It contains all the dependencies needed to install Docker. On top of Docker, it gives you a graphical interface to manage your images and containers.
Docker Desktop is proprietary software. It requires a paid per-user subscription for organizations with more than 250 employees or more than 10 million dollars in annual revenue. If you plan to use Docker Desktop for personal use, you do not need to pay for a license.

* Go to the official Docker website to download the Docker Desktop installer: https://docs.docker.com/desktop/install/windows-install/

* Once the download is complete, run the installer. You need administrator privileges on the PC. Note that Docker is installed for all users of the PC.

* You get a summary of what the installer will do. Make sure to check the *Use WSL 2 instead of Hyper-V* box so that Docker is installed using WSL 2.

* The installation begins, Docker Desktop extracts the files needed for the installation.

* Expect around 3 to 5 minutes depending on how fast your PC is.

If you see the success screen, the installation went well. You can close the installer and then restart your computer by clicking the *Close and restart* button.

* You can launch Docker Desktop from the shortcut added to your desktop, or search for Docker by pressing the Windows key and typing "Docker".

Docker Desktop will ask you to accept the software's terms of use. Then a page opens asking you to sign in to your Docker account — the same account as Docker Hub. But you are not required to sign in; click *Continue without signing in* to use Docker without logging in.

# Conclusion

In this article, you learned how to install Docker on a Windows 10/11 PC via Docker Desktop.

# Sources
Install Docker desktop on Windows. (2023, August 8). Docker Documentation. https://docs.docker.com/desktop/install/windows-install/
