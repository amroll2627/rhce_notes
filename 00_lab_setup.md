# RHCE Home Lab Setup

[Back to index](README.md)

## 1. Purpose

I built this lab to practise Ansible on several servers.

Baseline snapshots provide a way back to a working setup without reinstalling everything.

## 2. Platform and storage

The lab uses VMware Fusion on a Mac mini with Apple silicon M4.

The virtual machines use RHEL 10.0 for ARM. The installer is:

```text
rhel-10.0-aarch64-dvd.iso
```

The lab uses the Data volume on an external SSD of about 500 GB.

The Data volume is 270 GB and had about 198 GiB free before the lab setup. The free space will be different after creating VMs and snapshots.

The other SSD volumes contain backups and work files.

## 3. Virtual machines

The lab has these machines:

1. workstation.lab.example.com: 172.16.218.9
2. servera.lab.example.com: 172.16.218.10
3. serverb.lab.example.com: 172.16.218.11
4. serverc.lab.example.com: 172.16.218.12
5. serverd.lab.example.com: 172.16.218.13

Workstation has a desktop for VS Code. The other four machines are managed servers.

To check a machine:

```bash
cat /etc/redhat-release
uname -m
hostname
ip -br a
```

## 4. Network

The VMs use Fusion's Share with my Mac network.

Network settings:

1. Network: 172.16.218.0/24
2. Gateway: 172.16.218.2
3. DNS: 172.16.218.2
4. Interface: ens160
5. DHCP range: 172.16.218.128 to 172.16.218.254

The static addresses sit outside the DHCP range.

These entries were added to /etc/hosts on the lab VMs, keeping the existing entries:

```text
172.16.218.9  workstation.lab.example.com workstation
172.16.218.10 servera.lab.example.com servera
172.16.218.11 serverb.lab.example.com serverb
172.16.218.12 serverc.lab.example.com serverc
172.16.218.13 serverd.lab.example.com serverd
```

These addresses belong to this homelab. Another Fusion installation may use a different subnet.

## 5. SSH and sudo

SSH connections to the managed servers use the student account.

An SSH key was created on workstation, and its public key was copied to each managed server.

The managed servers allow student to use sudo without a password. The lab sudoers file was edited using:

```bash
sudo visudo -f /etc/sudoers.d/90-student-lab
```

The file contains:

```text
student ALL=(ALL) NOPASSWD: ALL
```

This gives student full sudo access on the lab VMs.

To verify access:

```bash
for node in servera serverb serverc serverd; do
  ssh -o BatchMode=yes \
      -o ConnectTimeout=5 \
      student@"${node}.lab.example.com" \
      'hostname; sudo -n whoami'
done
```

Each server returned its hostname and root.

## 6. Development environment

VS Code runs on workstation with the Ansible and Dev Containers extensions.

The project folder on workstation is:

```text
/home/student/rhce-lab
```

Inside the development container, the project is available at:

```text
/workspaces/rhce-lab
```

Podman is the container engine.

The development container image is:

```text
registry.redhat.io/ansible-automation-platform-25/ansible-dev-tools-rhel8:latest
```

The execution environment image is:

```text
registry.redhat.io/ansible-automation-platform-25/ee-supported-rhel8:latest
```

The images use RHEL 8 as their base. The VMs run RHEL 10. These are different parts of the setup.

The Podman image store inside the development container was empty at first. Pulling an image on workstation did not make it appear in that container's image store.

## 7. SSH agent

Before opening VS Code, run these commands as student in a workstation desktop terminal:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
code ~/rhce-lab
```

Then select Dev Containers: Reopen in Container.

Inside the development container, check the available key with:

```bash
ssh-add -l
```

## 8. Project files

The project uses four main files:

1. inventory: the managed servers and groups.
2. ansible.cfg: Ansible connection and sudo settings.
3. ansible-navigator.yml: Navigator and execution environment settings.
4. check-lab.yml: a basic connectivity test.

The configuration files are explained in the Chapter 1 notes.

Inventory contents:

```ini
[lab]
servera.lab.example.com ansible_host=172.16.218.10
serverb.lab.example.com ansible_host=172.16.218.11
serverc.lab.example.com ansible_host=172.16.218.12
serverd.lab.example.com ansible_host=172.16.218.13
```

## 9. Timezone problem

Ansible Navigator initially failed with:

```text
ModuleNotFoundError: No module named 'tzdata'
ZoneInfoNotFoundError: 'No time zone found with key UTC'
```

The tzdata RPM was installed, but /usr/share/zoneinfo/UTC was missing.

Installing the Python tzdata package alone did not fix the Navigator run.

This worked inside the development container:

```bash
microdnf reinstall -y tzdata
/usr/bin/python3.12 -sP -c 'from zoneinfo import ZoneInfo; print(ZoneInfo("UTC"))'
```

The check returned UTC.

To apply this fix when creating a new development container, this property can be added to the existing devcontainer.json:

```json
"postCreateCommand": "test -r /usr/share/zoneinfo/UTC || microdnf reinstall -y tzdata"
```

This is one property, not a complete JSON file. The surrounding commas must be correct.

## 10. Verify the lab

The check-lab.yml file contains:

```yaml
---
- name: Verify lab connectivity and privilege escalation
  hosts: lab
  gather_facts: false
  tasks:
    - name: Verify Ansible can execute Python through sudo
      ansible.builtin.ping:
```

Run it inside the development container from the project directory:

```bash
ansible-navigator run check-lab.yml
```

All four managed servers returned:

```text
ok=1 changed=0 unreachable=0 failed=0
```

The test was succesful.

The Ansible ping module checks connectivity and Python execution. It is not an ICMP network ping.

Because the configuration enables become, this test also checks execution through sudo.

There were deprecation warnings, but they did not fail this test.

## 11. Baseline snapshots

After the connectivity test passed, I saved snapshots of workstation and all four managed servers.

Before restoring a snapshot, save any newer notes or playbooks that need to be kept.

For another study session:

1. Start the VMs.
2. Start the SSH agent and load the key on workstation.
3. Open the project in VS Code.
4. Reopen the development container.
5. Run the connectivity test if needed.
6. Start the exercise.

This is a homelab. The official classroom has additional services and scripts. Commands such as lab start and lab grade are not included in a normal RHEL installation.
