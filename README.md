# My RHCE Notes

RHCE study notes, home lab setup and things to revisit during revision.

I am learning Ansible using the AU294 material for Ansible Automation Platform 2.5. These notes collect the topics, exercises and useful reminders in one place.

I use ChatGPT to help explain topics, review my attempts and draft these notes. The lab exercises and recorded results are from my own practice.

## Notes

1. [Home lab setup](00_lab_setup.md)
2. [Chapter 1: Introducing Ansible](01_chapter_one.md)
3. [Chapter 2: Introduction to Developing Automation Content](02_chapter_two.md)
4. [Chapter 3: Developing Automation Content: Variables](03_chapter_three.md)
5. [Chapter 4: Developing Automation Content: Task Control](04_chapter_four.md)
6. [Chapter 5: Developing Automation Content: Deploying Files](05_chapter_five.md)

## Lab

The lab runs on VMware Fusion on a Mac mini M4.

The lab has five RHEL 10 ARM virtual machines:

1. workstation
2. servera
3. serverb
4. serverc
5. serverd

VS Code, Podman and Ansible Navigator run on workstation.

## Learning progress

The basic lab setup is working. Ansible can connect to all four managed servers and run tasks through sudo.

Baseline snapshots were saved before starting the exercises.

Some settings were copied during the first setup. I still need to understand them and practise writing them myself.

Chapter 2 home lab exercises are complete. The notes cover inventories, playbooks, services, firewall rules and troubleshooting, including the final MariaDB and PHP web server review.

Chapter 3 home lab exercises are complete. The notes cover variables, facts, magic variables and Ansible Vault. The final HTTPS review returned the correct page with authentication, rejected access without login, and completed a repeated run with no changes. I still need practice recalling the options without checking earlier examples.

Chapter 4 home lab practice is complete for loops, conditions, handlers and error handling. The combined exercise recovered from a missing readiness file, then completed a repeated run with no changes. The notes include the mistakes with boolean comparisons and registered loop results. I still need to revisit this chapter and practise writing the combined playbook with less help.

Chapter 5 home lab practice is complete for file management and Jinja2 templates. The combined exercise produced the required configuration using variables, facts, filters, a condition and a loop, then completed a repeated run with no changes. The notes include the conflicting copy/lineinfile tasks, the vars warning and output corrections. The official guided exercises and chapter lab were not run during this session.

These are personal learning notes, not official Red Hat documentation.
