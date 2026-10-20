# Chapter 1: Introducing Ansible

[Back to index](README.md)

## 1. Why use Ansible?

Doing the same task manually on many servers takes time. It is also easy to miss a step or configure one server differently.

With Ansible, I can describe the work in a file and run it against several servers.

Examples include installing packages, creating users, updating configuration files and starting services.

I can keep these files in Git so I can review changes and use them again.

## 2. How Ansible works

For my Linux lab, Ansible connects to the managed servers using SSH.

Most Linux modules also need Python on the managed servers.

Ansible is agentless. I do not need a permanent Ansible agent running on each server. Working access and the correct permissions are still required.

Workstation hosts my automation tools. Servera through serverd are the managed nodes.

## 3. Important terms

1. Inventory: a list of hosts and groups.
2. Module: code that performs a specific operation.
3. Task: an instruction that calls a module.
4. Play: connects a set of hosts with tasks and settings.
5. Playbook: a YAML file containing one or more plays.
6. Role: a way to organise reusable automation content.
7. Collection: a package that can contain modules, plugins and roles.

For example, ansible.builtin.ping is a fully qualified module name.

Ansible Core provides the main automation engine. Ansible Automation Platform includes more tools and supported components.

Their version numbers are different. AAP 2.5 does not mean Ansible Core 2.5.

## 4. Idempotence

An idempotent task leaves the server in the requested state.

For example, a task can ask for a package to be installed. The first run may install it. The next run should report no change if the requested state is already correct.

Not every task is automatically idempotent. A shell command can repeat its action every time.

## 5. Development container and execution environment

These have seperate jobs.

The development container contains tools for writing and checking automation.

The execution environment contains Ansible and the dependencies used to run the automation.

An image is the packaged software used to create a container. Podman is the engine that manages containers.

In my setup:

1. I work in VS Code inside the development container.
2. I run Ansible Navigator.
3. Navigator uses Podman to start the execution environment.
4. Ansible connects to the managed servers through SSH.
5. Tasks use sudo when privilege escalation is enabled.

## 6. My ansible.cfg

This file provides default settings for Ansible.

It uses INI format:

```ini
[defaults]
inventory = ./inventory
remote_user = student
ask_pass = false

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false
```

What the settings mean:

1. inventory selects my project inventory file.
2. remote_user sets student as the default SSH user.
3. ask_pass disables the SSH password prompt.
4. become enables privilege escalation.
5. become_method selects sudo.
6. become_user selects root as the target user.
7. become_ask_pass disables the sudo password prompt.

Disabling a password prompt does not configure SSH keys or grant sudo access. I configured those separately.

Logging in as student and becoming root are two different steps.

## 7. Which ansible.cfg is used?

Ansible checks these locations in order:

1. The file specified by ANSIBLE_CONFIG.
2. ansible.cfg in the current directory.
3. ~/.ansible.cfg.
4. /etc/ansible/ansible.cfg.

It uses the first applicable file. It does not merge all four files together.

Ansible skips automatic loading of a current directory configuration if that directory is world writable.

Useful checks:

```bash
ansible --version
ansible-config dump --only-changed
```

The first command shows the configuration file path. The second shows settings that differ from the defaults.

These checks describe the environment where I run them. An execution environment can have different files or software from the development container.

## 8. My ansible-navigator.yml

This file controls how Ansible Navigator runs.

It uses YAML format:

```yaml
---
ansible-navigator:
  execution-environment:
    enabled: true
    container-engine: podman
    image: registry.redhat.io/ansible-automation-platform-25/ee-supported-rhel8:latest
    pull:
      policy: missing
  mode: stdout
  playbook-artifact:
    enable: false
```

What the settings mean:

1. enabled: true tells Navigator to use an execution environment.
2. container-engine: podman selects Podman.
3. image selects the execution environment image.
4. policy: missing downloads the image if it is not already available in that image store.
5. mode: stdout prints the output in the terminal.
6. playbook-artifact enable: false disables the Navigator playbook artifact file.

Disabling the playbook artifact does not disable every type of logging.

The latest image tag can change over time. With the missing pull policy, an existing local image is not automatically refreshed on every run.

## 9. Running the test

From the project directory:

```bash
ansible-navigator run check-lab.yml
```

The configuration files are useful for later playbooks too. The check-lab.yml file is specifically for checking the basic lab connection.

For the baseline test, every server returned:

```text
ok=1 changed=0 unreachable=0 failed=0
```

This confirms the basic connection and sudo setup. It does not test every possible Ansible task.

## 10. Git and GitHub

Git tracks changes to files. GitHub hosts a remote repository and provides a website for working with it.

The basic terms are:

1. Repository: the project and its history.
2. Clone: create a local copy of a remote repository.
3. Stage: select changes for the next commit.
4. Commit: record changes with a message.
5. Push: send local commits to the remote repository.
6. Pull: fetch and integrate remote changes.
7. Branch: a line of development in the repository.

For now, I can write and edit my notes through GitHub.

Later, inside a local clone, a normal update could look like this:

```bash
git status
git add 01_chapter_one.md
git commit -m "Update Chapter 1 notes"
git push
```

A local commit is not automatically uploaded. Push sends it to GitHub.

## 11. What I need to practise

1. Explain how Ansible connects to a managed server.
2. Understand the difference between the development container and execution environment.
3. Write a basic ansible.cfg.
4. Explain the engine, image and pull policy in ansible-navigator.yml.
5. Check which configuration file is being used.
6. Verify SSH and sudo access.
7. Run a playbook and read its result.
8. Save changes in Git.

I do not need to memorise my lab's long image address. I need to understand how to use the image and settings required by a task.

These configuration files will appear again when I practise inventories, playbooks and later topics.
