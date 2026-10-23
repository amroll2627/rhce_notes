# My RHCE Notes

This is where I keep my RHCE study notes and home lab setup.

I am learning Ansible using the AU294 material for Ansible Automation Platform 2.5. I write these notes in my own words so I can refer to them when I forget something.

## My notes

1. [Home lab setup](00_lab_setup.md)
2. [Chapter 1: Introducing Ansible](01_chapter_one.md)
3. [Chapter 2: Introduction to Developing Automation Content](02_chapter_two.md)

## My lab

I use VMware Fusion on a Mac mini M4.

The lab has five RHEL 10 ARM virtual machines:

1. workstation
2. servera
3. serverb
4. serverc
5. serverd

I use VS Code, Podman and Ansible Navigator on workstation.

## My learning progress

The basic lab setup is working. Ansible can connect to all four managed servers and run tasks through sudo.

I have saved baseline snapshots before starting the exercises.

Some settings were copied during the first setup. I still need to understand them and practise writing them myself.

I have completed my Chapter 2 home lab exercises. My notes cover inventories, playbooks, services, firewall rules and troubleshooting, including the final MariaDB and PHP web server review.

These are my personal learning notes, not official Red Hat documentation.
