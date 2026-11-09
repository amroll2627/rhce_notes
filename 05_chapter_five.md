# Chapter 5: Developing Automation Content: Deploying Files

[Back to index](README.md)

Status: Home lab practice completed for file management and Jinja2 templates. The final combined exercise produced the requested configuration and completed a repeated run with no changes.

These notes use AU294 for AAP 2.5 with the RHEL 10 ARM home lab. Variables and facts are in [Chapter 3](03_chapter_three.md). Task loops and conditions are in [Chapter 4](04_chapter_four.md).

I needed the complete Jinja syntax before the examples made sense. Using variables, filters, a loop and a condition together made the template flow much clearer.

The examples follow our practice, with task names and spacing cleaned up where useful. **Observed** means a result recorded during the lab. These were adapted home lab exercises. The official guided exercises in sections 5.2 and 5.4 and the official lab in 5.5 were not run during this session.

## 1. Chapter map and lab commands

| Section | Topic |
| --- | --- |
| 5.1 and 5.2 | Automating file management tasks, followed by a guided exercise |
| 5.3 and 5.4 | Jinja2 templates and filters, followed by a guided exercise |
| 5.5 and 5.6 | Combined chapter lab and summary |

The exercises targeted `servera` using `inventory-bab2-2`. This is the same home lab server at `172.16.218.10`.

```bash
ansible-navigator run bab5-files.yml -i inventory-bab2-2 -m stdout
ansible-navigator run bab5-final.yml -i inventory-bab2-2 -m stdout
```

Reference syntax check:

```bash
ansible-navigator run bab5-final.yml -i inventory-bab2-2 -m stdout --syntax-check
```

## 2. Choosing a file module

| Requirement | Module |
| --- | --- |
| Create a directory or manage file attributes | ansible.builtin.file |
| Copy an existing file from the controller | ansible.builtin.copy with src |
| Write complete literal file content from a playbook | ansible.builtin.copy with content |
| Ensure or replace an individual line | ansible.builtin.lineinfile |
| Manage a group of lines between markers | ansible.builtin.blockinfile |
| Generate a file using variables and facts | ansible.builtin.template |

The source of a normal copy or template task is on the controller side. The destination is on the managed host.

## 3. Directory creation and copy

The first exercise used a play with `hosts: servera` and `become: true`:

```yaml
- name: Create directory
  ansible.builtin.file:
    path: /tmp/rhce-chapter5
    state: directory
    mode: '0755'
    owner: root
    group: root

- name: Create file
  ansible.builtin.copy:
    dest: /tmp/rhce-chapter5/info.txt
    content: "Managed by Ansible"
    mode: '0644'
    owner: root
    group: root
```

**Observed:** Both tasks changed on the creation run. The repeated run reported `ok=3 changed=0 failed=0`, including fact gathering.

The directory comes first because the following task needs its parent directory. Modes are quoted strings.

The next task copied a local file:

```yaml
- name: Copy file
  ansible.builtin.copy:
    src: files/chapter5-message.txt
    dest: /tmp/rhce-chapter5/message.txt
    owner: root
    group: root
    mode: '0644'
```

Create `files/chapter5-message.txt` in the controller project before running this task. Its exact sentence was not included in the recorded output; any simple text is enough to repeat the exercise.

**Observed:** The new copy task changed once. The repeated play reported `ok=4 changed=0 failed=0`.

## 4. Lineinfile and the wrong path

The exercise was to add `Chapter 5 file management practice` to info.txt while keeping the original line.

My first attempt used `/tmp/rhce-chapter5-info.txt` instead of `/tmp/rhce-chapter5/info.txt`. With `create: true`, lineinfile created the unintended file.

**Observed:** Even that wrong-path task reached `changed=0` on the next run. An unchanged recap does not prove that the path matches the requirement.

Lineinfile needs `create: true` to create a missing target file. Without it, a missing file causes failure. The module also accepts `dest` as an alias for `path`.

## 5. Copy and lineinfile conflicting

After fixing the path, two tasks managed info.txt:

1. Copy wrote only `Managed by Ansible`.
2. Lineinfile added the chapter practice line.
3. On the next run, copy removed the extra line, then lineinfile added it again.

**Observed:** Both tasks reported changed on every repeated run, with `changed=2`.

The correction was to manage both lines using lineinfile:

```yaml
- name: Ensure the first line exists
  ansible.builtin.lineinfile:
    path: /tmp/rhce-chapter5/info.txt
    line: Managed by Ansible
    create: true
    owner: root
    group: root
    mode: '0644'

- name: Ensure the practice line exists
  ansible.builtin.lineinfile:
    path: /tmp/rhce-chapter5/info.txt
    line: Chapter 5 file management practice
```

The first corrected attempt omitted create and passed only because the file existed. We added create and tested after removing info.txt.

**Observed:** Both line tasks changed when rebuilding the file. The next run reported `ok=5 changed=0 failed=0` for the play at that stage.

Multiple lineinfile tasks or a loop can manage several lines. Blockinfile is useful when those lines form one unit.

## 6. Adding a managed block

```yaml
- name: Insert a block in file
  ansible.builtin.blockinfile:
    path: /tmp/rhce-chapter5/app.conf
    block: |
      APP_NAME=rhce-lab
      APP_ENV=practice
    create: true
    owner: root
    group: root
    mode: '0644'
```

The YAML `|` preserves line breaks. The observed content was:

```text
# BEGIN ANSIBLE MANAGED BLOCK
APP_NAME=rhce-lab
APP_ENV=practice
# END ANSIBLE MANAGED BLOCK
```

The markers identify the block being managed. Content outside the block can remain in the file.

**Observed:** The block changed on its first run, then reported no change. After adding explicit attributes, ls showed root ownership, root group and mode 0644.

These values were literal text. Changing the later app_env variable did not change the hardcoded APP_ENV line in this separate file.

## 7. Owner, group and mode

Follow the requested final state. Do not copy ownership and permissions from a previous task without checking the current requirement.

The earlier directory and file exercises requested root ownership and specific modes. Adding those attributes to the block task was a lab choice, not a rule for every task.

For the modules used here, an omitted mode keeps an existing destination's mode; a new file's mode depends on the system umask. Omitted ownership depends on the executing user and existing file. Attributes are not inherited from a previous task.

The final exercise did not request particular attributes, so its recorded playbook omitted them. Necessary access still has to be available. This is a way to read requirements, not a claim about undisclosed exam marking rules.

## 8. Variables need names

The first template attempt put unnamed values under vars. A later attempt used a list of dictionaries:

```yaml
vars:
  - app_name: rhce-lab
  - app_env: practice
```

**Observed:** The lab version accepted this list but produced a deprecation warning. The correction was a dictionary without the list markers:

```yaml
vars:
  app_name: rhce-lab
  app_env: practice
```

App_name is the variable name. Rhce-lab is its value. Tasks remains a list with `- name:` entries.

## 9. Rendering the first template

Controller file `templates/app.conf.j2`:

```jinja2
APP_NAME={{ app_name }}
APP_ENV={{ app_env }}
HOSTNAME={{ ansible_facts['hostname'] }}
```

Playbook task:

```yaml
- name: Create app template file
  ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /tmp/rhce-chapter5/app-template.conf
```

**Observed destination:**

```text
APP_NAME=rhce-lab
APP_ENV=practice
HOSTNAME=servera
```

Ansible renders the template on the controller using variables available for the target host. The hostname came from facts gathered at the beginning of the play. The source keeps its expressions; the destination contains their results.

The first template had no final newline, so the terminal displayed an end-of-line marker next to the final value. Saving a newline at the end of the .j2 file fixed the display and caused one expected content change.

```bash
ssh student@servera "cat /tmp/rhce-chapter5/app-template.conf"
ssh student@servera "ls -l /tmp/rhce-chapter5/app.conf"
```

An earlier `ssh student@servera -c 'cat ...'` failed with `Unknown cipher type`. In ssh, -c selects a cipher. Put the remote command after the host as shown above.

## 10. Filters: upper and default

```jinja2
APP_ENV={{ app_env | upper }}
APP_PORT={{ app_port | default(8080) }}
```

The pipe passes a value to a filter. It is not a shell pipeline.

| Input tested | Observed output |
| --- | --- |
| app_env: practice | APP_ENV=PRACTICE |
| No app_port variable | APP_PORT=8080 |
| app_port: 9090 | APP_PORT=9090 |

Upper changes the rendered value, leaving the original variable unchanged. Default supplies 8080 when app_port is undefined. This form does not replace every empty or false value.

The repeated run after the uppercase edit reported `changed=0`.

## 11. Looping inside a template

The vars dictionary included:

```yaml
app_users:
  - student
  - developer
```

Complete `templates/user.conf.j2`:

```jinja2
{% for user in app_users %}
ALLOWED_USER={{ user }}
{% endfor %}
```

Task:

```yaml
- name: Create user template file
  ansible.builtin.template:
    src: templates/user.conf.j2
    dest: /tmp/rhce-chapter5/user.conf
```

| Part | Meaning |
| --- | --- |
| for user in app_users | Take each entry from the list |
| user | Temporary name for the current entry |
| {{ user }} | Print the current entry |
| endfor | End the loop |

The first attempt had a space in `ALLOWED_USER= {{ user }}`. That space appeared in the result. Removing it gives:

```text
ALLOWED_USER=student
ALLOWED_USER=developer
```

A task loop repeats a task. A Jinja loop repeats text while rendering one file. This example does not create Linux user accounts.

## 12. Conditions inside a template

```jinja2
{% if app_env == 'practice' %}
DEBUG=true
{% else %}
DEBUG=false
{% endif %}
```

Use the variable directly in the condition, without extra `{{ }}`. The `==` operator compares values.

**Both branches were tested:**

| app_env | Rendered APP_ENV with upper | DEBUG |
| --- | --- | --- |
| practice | PRACTICE | true |
| production | PRODUCTION | false |

The upper filter on APP_ENV does not change the original app_env variable. The condition still compares it with lowercase practice.

## 13. Complete practice playbook

This assembles the corrected `bab5-files.yml` using production from the final condition test:

```yaml
---
- name: Chapter 5 intro
  hosts: servera
  become: true
  vars:
    app_name: rhce-lab
    app_env: production
    app_port: 9090
    app_users:
      - student
      - developer
  tasks:
    - name: Create directory
      ansible.builtin.file:
        path: /tmp/rhce-chapter5
        state: directory
        mode: '0755'
        owner: root
        group: root

    - name: Ensure the first line exists
      ansible.builtin.lineinfile:
        path: /tmp/rhce-chapter5/info.txt
        line: Managed by Ansible
        create: true
        owner: root
        group: root
        mode: '0644'

    - name: Copy file
      ansible.builtin.copy:
        src: files/chapter5-message.txt
        dest: /tmp/rhce-chapter5/message.txt
        owner: root
        group: root
        mode: '0644'

    - name: Ensure the practice line exists
      ansible.builtin.lineinfile:
        path: /tmp/rhce-chapter5/info.txt
        line: Chapter 5 file management practice

    - name: Insert a block in file
      ansible.builtin.blockinfile:
        path: /tmp/rhce-chapter5/app.conf
        block: |
          APP_NAME=rhce-lab
          APP_ENV=practice
        create: true
        owner: root
        group: root
        mode: '0644'

    - name: Create app template file
      ansible.builtin.template:
        src: templates/app.conf.j2
        dest: /tmp/rhce-chapter5/app-template.conf

    - name: Create user template file
      ansible.builtin.template:
        src: templates/user.conf.j2
        dest: /tmp/rhce-chapter5/user.conf
```

Complete `templates/app.conf.j2`:

```jinja2
APP_NAME={{ app_name }}
APP_ENV={{ app_env | upper }}
HOSTNAME={{ ansible_facts['hostname'] }}
APP_PORT={{ app_port | default(8080) }}
{% if app_env == 'practice' %}
DEBUG=true
{% else %}
DEBUG=false
{% endif %}
```

Use the user template from section 11 and the local message file from section 3. Save a final newline in both templates.

App.conf and app-template.conf are separate files, so blockinfile and template do not overwrite each other's work.

## 14. Final combined home lab

The final exercise combined all the practiced Jinja features in one template.

Recorded `bab5-final.yml`, with task names cleaned up:

```yaml
---
- name: Chapter 5 final
  hosts: servera
  vars:
    app_name: rhce-lab
    app_env: production
    app_users:
      - student
      - developer

  tasks:
    - name: Create directory
      ansible.builtin.file:
        path: /tmp/rhce-ch5-final
        state: directory

    - name: Create configuration file
      ansible.builtin.template:
        src: templates/ch5final.conf.j2
        dest: /tmp/rhce-ch5-final/app.conf
```

App_port is deliberately absent. This play keeps the lab's existing connection and privilege configuration. It does not explicitly set become or file attributes.

Corrected `templates/ch5final.conf.j2`:

```jinja2
APP_NAME={{ app_name }}
APP_ENV={{ app_env | upper }}
HOSTNAME={{ ansible_facts['hostname'] }}
APP_PORT={{ app_port | default(8080) }}
{% if app_env == 'practice' %}
DEBUG=true
{% else %}
DEBUG=false
{% endif %}
{% for user in app_users %}
ALLOWED_USER={{ user }}
{% endfor %}
```

The first attempt was idempotent but had two content mistakes:

| First attempt | Correction |
| --- | --- |
| APP_ENV=production | Add upper to produce PRODUCTION |
| APP_Port=8080 | Correct the literal key to APP_PORT |

Spacing inside expressions is for readability. Literal text and spaces outside expressions become part of the file. An unchanged recap alone does not prove that the requested text is correct.

## 15. Recorded final verification

```bash
ansible-navigator run bab5-final.yml -i inventory-bab2-2 -m stdout
ssh student@servera "cat /tmp/rhce-ch5-final/app.conf"
ansible-navigator run bab5-final.yml -i inventory-bab2-2 -m stdout
```

**Observed content after correction:**

```text
APP_NAME=rhce-lab
APP_ENV=PRODUCTION
HOSTNAME=servera
APP_PORT=8080
DEBUG=false
ALLOWED_USER=student
ALLOWED_USER=developer
```

The correction run changed only the template task. The directory already existed, so this was not a completely fresh first run.

**Observed final repeated-run recap:**

```text
servera : ok=3 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

The three successful tasks were fact gathering, directory management and template deployment. The content matched the requirement and the repeated run made no changes.

## 16. Revision reminders

| Requirement or symptom | Check |
| --- | --- |
| Write a complete file | copy or template |
| Manage one line while preserving other content | lineinfile |
| Manage a group of lines | blockinfile |
| Create a missing lineinfile/blockinfile target | create: true |
| Same file changes on every run | Conflicting desired content between tasks |
| Print a variable or fact | {{ expression }} |
| Control generated text | {% for %} or {% if %}, with the matching end statement |
| Variable may be undefined | default filter |
| Vars warning about a list | Use a dictionary under vars |
| Successful run but wrong output | Exact path, key names, values and whitespace |

```bash
ansible-doc ansible.builtin.file
ansible-doc ansible.builtin.copy
ansible-doc ansible.builtin.lineinfile
ansible-doc ansible.builtin.blockinfile
ansible-doc ansible.builtin.template
```

Module documentation explains options such as src and dest. The Jinja template language reference explains for/endfor and if/else/endif.

For revision, write the final playbook and template again from the required output. I still need practice recalling the full Jinja syntax without checking the earlier examples.

## 17. References

1. AU294 AAP 2.5, Chapter 5: Developing Automation Content: Deploying Files. The examples here record adapted home lab practice.
2. [File module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/file_module.html).
3. [Copy module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html).
4. [Lineinfile module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/lineinfile_module.html).
5. [Blockinfile module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/blockinfile_module.html).
6. [Template module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/template_module.html).
7. [Using variables](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html).
8. [Jinja template designer documentation](https://jinja.palletsprojects.com/en/stable/templates/).

[Back to index](README.md)
