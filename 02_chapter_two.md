# Chapter 2: Introduction to Developing Automation Content

[Back to index](README.md)

Status: Adapted Chapter 2 home lab exercises completed. This is a recap for revision and for anyone following the same topics.

These exercises use the AU294 material for AAP 2.5 on a RHEL 10 ARM lab. The basic configuration is covered in [Chapter 1](01_chapter_one.md).

These notes follow the exercises completed in the lab. Some task names and formatting have been cleaned up in the examples. Blocks marked **Observed** are shortened lab results. **Reference** commands are reminders; not every option shown was tested.

## 1. Topics covered

1. Build inventories with hosts, groups, child groups, ranges and variables.
2. Write YAML playbooks and run them with Ansible Navigator.
3. Understand the SSH user and the user that runs a task.
4. Read task results and check idempotence.
5. Use multiple plays for different groups of servers.
6. Manage packages, files, services and firewall rules.
7. Find mistakes with syntax checks, lint and verbose output.
8. Set up MariaDB on serverc and Apache with PHP on serverb, then test HTTP from servera.

The home lab version of the exercises is complete. The Red Hat classroom lab grader was not run. The final PHP page was a simple test page; no database user was configured and no PHP connection to MariaDB was tested.

## 2. First grouped inventory

The `inventory-bab2` file used two groups under a parent group:

```ini
[webservers]
servera ansible_host=172.16.218.10
serverb ansible_host=172.16.218.11

[dbservers]
serverc ansible_host=172.16.218.12
serverd ansible_host=172.16.218.13

[production:children]
webservers
dbservers
```

1. `servera` is the inventory name used in plays and output.
2. `ansible_host` tells Ansible which address to connect to.
3. `[production:children]` contains group names. A normal `[production]` section contains hosts.
4. A host can belong to several groups. Repeating the same inventory name does not create another unique host.
5. `all` contains every inventory host. `ungrouped` contains hosts that are not members of another group.

I first wrote `:webservers` and `:dbservers` under `[production]`. Those entries were treated as host names. The `:children` suffix belongs on the group heading.

Adding serverb to another group still leaves four unique hosts in this inventory. This depends on the inventory name. Two different aliases pointing to the same IP are still two inventory hosts.

Using an IP in `ansible_host` means Ansible does not need to resolve the alias through DNS or `/etc/hosts` for that connection. It does not rename the remote operating system or change how other programs resolve names.

## 3. Inspecting inventory and variables

Commands used to inspect the inventory:

```bash
ansible-navigator inventory -i inventory-bab2 -m stdout --graph
ansible-navigator inventory -i inventory-bab2 -m stdout --graph production
ansible-navigator inventory -i inventory-bab2 -m stdout --host serverb
ansible-navigator inventory -i inventory-bab2 -m stdout --list
```

1. `inventory` is the Navigator subcommand. I once forgot it and got an invalid choice error.
2. `-i` selects the inventory source for this command.
3. `-m stdout` prints the result in the terminal.
4. `--graph` shows group relationships and membership.
5. `--host` shows inventory variables for one host. It does not gather live facts from that machine.
6. `--list` shows the parsed inventory and host variables as structured output.

The configured default inventory is `./inventory`. The file is not automatically called `inventory.ini`. Specifying `-i inventory-bab2` selects that file instead of the configured default.

The `inventory-vars` file was a copy of the grouped inventory above with these additions:

```ini
[production:vars]
ansible_user=student
lab_env=homelab

[webservers:vars]
app_port=8080
```

```bash
ansible-navigator inventory -i inventory-vars -m stdout --host servera
ansible-navigator inventory -i inventory-vars -m stdout --host serverc
```

**Observed:** Both hosts had `ansible_user` and `lab_env`. Only servera had `app_port` because servera belongs to `webservers`, while serverc belongs to `dbservers`.

The variables from the parent group apply to hosts in its child groups. A variable such as `app_port` only stores data. Defining it does not open a port or configure an application by itself.

The `inventory-range` test used:

```ini
[lab]
server[a:d]

[odd_nodes]
node[01:07:2]
```

```bash
ansible-navigator inventory -i inventory-range -m stdout --graph
```

**Observed:** The ranges expanded to servera through serverd, and node01, node03, node05 and node07. The leading zero was kept. This inventory check did not prove that the example node hosts existed or were reachable.

These were static inventories. Dynamic inventory plugins can obtain hosts from another system, but building one was outside these exercises.

## 4. Inventory for the earlier playbook exercises

Most of the earlier playbooks used `inventory-bab2-2`:

```ini
[webservers]
servera ansible_host=172.16.218.10
serverb ansible_host=172.16.218.11
serverc ansible_host=172.16.218.12
serverd ansible_host=172.16.218.13

[raleigh]
servera
serverb

[mountainview]
serverc

[london]
serverd

[production]
serverc
serverd

[development]
servera

[testing]
serverb

[us:children]
raleigh
mountainview
```

**Observed:** `--graph us` showed servera, serverb and serverc. `--graph ungrouped` was empty because all four hosts belonged to named groups.

Group membership depends on the selected inventory. In this file, `production` contains only serverc and serverd. In the first inventory, it included all four hosts through child groups.

## 5. First playbook and YAML layout

This is the later version of `first-play.yml`, with fact gathering disabled:

```yaml
---
- name: Create my practice directory
  hosts: testing
  become: true
  gather_facts: false
  tasks:
    - name: Ensure the practice directory exists
      ansible.builtin.file:
        path: /tmp/rhce-bab2
        state: directory
        owner: root
        group: root
        mode: '0755'
```

1. A playbook is a list of plays. The first `- name` starts a play.
2. `hosts` selects the target hosts from inventory. Here, `testing` means serverb.
3. `tasks` contains the list of actions for that play.
4. `ansible.builtin.file` is the module's fully qualified collection name, or FQCN.
5. `path`, `state`, `owner`, `group` and `mode` are parameters under that module.
6. `state: directory` makes sure the directory exists. A task name containing the word "check" does not make the task read only.

Use spaces for indentation and quote permissions such as `'0755'` and `'0644'`.

My first attempts had a missing colon after the module name, then parameters at the wrong indentation. The first problem caused a YAML error. The second caused `conflicting action statements: ansible.builtin.file, path` because Ansible could not read the task structure correctly.

```bash
ansible-navigator run first-play.yml -i inventory-bab2-2 -m stdout --syntax-check
ansible-navigator run first-play.yml -i inventory-bab2-2 -m stdout
```

`--syntax-check` checks the playbook without running its tasks. It does not prove that a source file exists or that the remote operations will succeed.

## 6. Reading the result and gathering facts

The original directory exercise used the default fact gathering. **Observed:**

```text
First run:  serverb : ok=2 changed=1 unreachable=0 failed=0
Second run: serverb : ok=2 changed=0 unreachable=0 failed=0
```

There were two successful tasks: automatic fact gathering and the directory task.

After disabling fact gathering, the unchanged directory run reported:

```text
serverb : ok=1 changed=0 unreachable=0 failed=0
```

| Field | Meaning |
| --- | --- |
| `ok` | Successful task results for that host, including tasks that changed something |
| `changed` | Successful results that reported a change; this is part of `ok` |
| `unreachable` | Ansible could not establish the required connection |
| `failed` | A task failed |
| `skipped` | A task was skipped |

The recap is per host. It is not a count of inventory groups. `ok=2 changed=1` does not mean three tasks ran.

`gather_facts: false` skips the automatic fact gathering task for that play. It does not remove the Python requirement for normal remote modules. Each play has its own settings; disabling facts in the first play does not disable them in the next play.

## 7. Check mode and correcting drift

```bash
ansible-navigator run first-play.yml -i inventory-bab2-2 -m stdout --check --diff
```

1. `--check` asks supported modules to predict changes without applying them.
2. `--diff` can show differences for modules that support it.
3. Check mode is not a complete test of every module or every runtime dependency.

In this exercise, the directory permission was changed to `0700` while the playbook requested `0755`.

**Observed:** Check mode reported `changed=1`, but the actual permission stayed at `0700`. After a normal run, it became `0755`.

```bash
ssh student@172.16.218.11 "stat -c '%a %U:%G %n' /tmp/rhce-bab2"
```

**Observed after the normal run:**

```text
755 root:root /tmp/rhce-bab2
```

This helped me understand idempotence. The playbook describes the required state, and the module corrects a difference when it finds one.

## 8. SSH user and become

I initially mixed up the login user and the user running the task. The lab normally connects through SSH as `student`, then uses sudo to become `root` when become is enabled.

For the settings compared here, the SSH user precedence is:

1. Inventory variable `ansible_user`.
2. Play keyword `remote_user`.
3. Configuration option `remote_user` in `ansible.cfg`.

This is the comparison used in the exercise, not the complete Ansible precedence list. Other variable sources can also override values.

**Reasoning example, not a tested account setup:** If configuration says student, the play says operator, and inventory sets `ansible_user=deploy`, SSH uses deploy. With become disabled, remote tasks run as deploy. With become enabled and `become_user=root`, remote tasks run as root through the configured become method.

`become_user: root` does not enable become by itself. Also, `become_ask_pass=false` does not supply a sudo password. The lab works because student already has passwordless sudo.

The file module's `owner` and `group` settings control the object being managed. They do not select the account running the task.

## 9. Multiple plays

The `multiple-plays.yml` exercise used different groups and privileges:

```yaml
---
- name: Create a directory owned by root
  hosts: raleigh
  become: true
  gather_facts: false
  tasks:
    - name: Ensure the root directory exists
      ansible.builtin.file:
        path: /tmp/rhce-root
        state: directory
        owner: root
        group: root
        mode: '0755'

- name: Create a directory owned by student
  hosts: production
  become: false
  gather_facts: false
  tasks:
    - name: Ensure the student directory exists
      ansible.builtin.file:
        path: /tmp/rhce-student
        state: directory
        owner: student
        group: student
        mode: '0755'
```

```bash
ansible-navigator run multiple-plays.yml -i inventory-bab2-2 -m stdout
```

The first play targeted servera and serverb. The second targeted serverc and serverd. The successful run completed the first play before moving to the second.

**Observed:** All four hosts reported `ok=1 changed=1` on the successful creation run, then `ok=1 changed=0` on the next run.

One earlier attempt failed with `file (/tmp/rhce-root) is absent, cannot continue`. I corrected the file task before the successful run. The final example explicitly requests `state: directory` so Ansible creates a missing directory. A syntax check alone did not catch the runtime problem.

## 10. Module behaviour and multiline content

The `module-behaviour.yml` exercise compared a command with a module that manages a file:

```yaml
---
- name: Compare command and copy behaviour
  hosts: testing
  become: false
  gather_facts: false
  tasks:
    - name: Check uptime
      ansible.builtin.command:
        cmd: /usr/bin/uptime

    - name: Write my practice note
      ansible.builtin.copy:
        content: |
          Managed by Ansible.
          Chapter 2 practice.
        dest: /tmp/rhce-note.txt
        owner: student
        group: student
        mode: '0644'
```

```bash
ansible-navigator run module-behaviour.yml -i inventory-bab2-2 -m stdout
ssh student@172.16.218.11 'cat /tmp/rhce-note.txt'
```

**Observed:** The run after changing the content reported `ok=2 changed=2`. The next run reported `ok=2 changed=1`. The remaining change came from `uptime`.

The command module normally reports changed when it executes a command. That report does not mean uptime modified the server. It also does not compare the new command output with the previous run. This example intentionally demonstrates the default reporting; it is not a template for a lint clean read only command task.

The copy task compares the requested content and managed attributes with the destination. It reported `ok` when they already matched.

The exercise started with `src: files/note.txt`, then changed to `content:`. With normal copy behaviour, `src` is a file available to Ansible on the controller side, including the project mounted into the execution environment. `dest` is on the managed host. Use either `src` or `content` for this task.

The `|` YAML style preserves line breaks. The `>` style folds ordinary adjacent lines into spaces. Blank lines and extra indentation have additional rules, so `>` does not remove every possible newline.

For choosing modules:

1. Prefer a module that describes the required state, such as `copy`, `file` or `dnf`.
2. `command` does not process shell operators such as pipes and output redirection.
3. `shell` is available when shell processing is needed, but repeated runs still need to be considered.

## 11. Running a command only when needed

The `command-once.yml` playbook:

```yaml
---
- name: Create a marker only when missing
  hosts: testing
  become: false
  gather_facts: false
  tasks:
    - name: Create the marker file
      ansible.builtin.command:
        cmd: /usr/bin/touch /tmp/rhce-once.marker
        creates: /tmp/rhce-once.marker
```

```bash
ansible-navigator run command-once.yml -i inventory-bab2-2 -m stdout
```

**Observed:** The first run reported `ok=1 changed=1`. The second reported `ok=1 changed=0`.

`creates` checks whether the path already exists. It does not create the file itself. The `touch` command creates it when execution is allowed. If the marker is removed, the next normal run can execute the command again.

The related `removes` option makes a command eligible to run when the matching path exists, subject to other guards such as `creates`. It does not delete the path by itself. There was no separate removes exercise in this lab.

## 12. Services, packages and the firewall mistake

The earlier `webserver.yml` installed Apache on the `raleigh` group, copied an HTML page, started and enabled httpd, and configured the firewall.

**Observed:** The first run reported `ok=5 changed=4` per host. The next run reported `ok=5 changed=0`. The fifth successful task was gathering facts.

I still could not reach the page with curl. My firewall task had only changed the permanent configuration. I added `immediate: true`:

```yaml
- name: Allow HTTP now and after a firewall reload
  ansible.posix.firewalld:
    service: http
    permanent: true
    immediate: true
    state: enabled
```

This is a task fragment to place inside a play's task list. Firewalld must be running for this runtime change.

**Observed:** Only the firewall task changed on the next run, and curl then returned the HTML page from both servera and serverb.

```bash
curl --max-time 5 http://172.16.218.10
curl --max-time 5 http://172.16.218.11
ssh student@172.16.218.10 "sudo ss -lntp 'sport = :80'"
```

The socket check showed httpd listening on port 80. The curl checks confirmed that the page was reachable.

Other useful details:

1. `state: present` ensures a package is installed. `state: latest` can also upgrade it when a newer version is available. The later exercises used present.
2. A service's `state: started` ensures it is running. `enabled: true` handles startup at boot. They are different settings.
3. `dnf.name` accepts a package list. `systemd_service.name` expects one unit, so different services use separate tasks.
4. `ansible.builtin.systemd` worked as a redirect. The final notes use the canonical `ansible.builtin.systemd_service` name.

## 13. Testing HTTP and saving the result

In `intranet.yml`, the first play configured servera and a second play on serverb tested it. **Observed:** The response had status 200 and content `Welcome to my RHCE intranet!` with a final newline.

The final review reversed those roles: serverb hosted the page and servera tested it.

```yaml
- name: Check the web page
  ansible.builtin.uri:
    url: http://172.16.218.11/index.php
    status_code: 200
    return_content: true
  register: web_result

- name: Show the returned content
  ansible.builtin.debug:
    var: web_result.content
```

These are task fragments for the test play.

1. `status_code` defines an accepted HTTP response code.
2. `return_content: true` asks for the response body in the result.
3. `register` saves the task result under the chosen variable name.
4. `register` is a task keyword. It lines up with the module name, not with `url`.
5. `web_result.content` selects the content field from that result. To inspect the whole result, use `var: web_result`.

I first put register inside the uri parameters and got `Unsupported parameters ... register`. After removing it, I got `VARIABLE IS NOT DEFINED!` because I had not saved the result. The corrected example above fixes both problems.

In the Ansible version used here, debug printed the undefined variable message while the task still appeared as ok. I had to read the output, not only look at `failed=0`.

I also used `verbosity: 2` in debug once. That task was skipped at normal verbosity. Running with `-vv`, or removing the verbosity condition, allowed it to display.

## 14. Checking and troubleshooting playbooks

The usual check sequence:

```bash
ansible-lint web-review.yml
ansible-navigator run web-review.yml -i inventory-review -m stdout --syntax-check
ansible-navigator run web-review.yml -i inventory-review -m stdout
```

Run the last command again to check repeated behaviour when the exercise calls for it.

Lint checks rules and common mistakes. Syntax checking validates playbook structure. A normal run checks what really happens with the selected hosts and files. Passing the first two does not guarantee the third will work.

Mistakes encountered during the exercises:

| What happened | Correction |
| --- | --- |
| Playbook could not be found | Create and save the file at the path passed to the command |
| YAML expected a colon | The module name needed its trailing colon |
| Conflicting action statements | Indent module parameters under the module |
| Missing `files/indx.html` | Correct the source path to the actual `files/index.html` |
| A list passed as a service name | Split httpd and firewalld into separate service tasks |
| `INVALID_SERVICE` mentioning `httpimm` | Correct the firewall service value to `http` |
| Unsupported uri parameter `register` | Move register out to task level |
| `VARIABLE IS NOT DEFINED!` | Save the result with register and use the same variable name |
| `--debug` or `-debug` rejected | Use verbosity flags such as `-vv` instead |
| `ansible-lint run ... -m stdout` rejected | Use `ansible-lint` to check and `ansible-navigator run` to execute |

During the source file exercise, lint and syntax checking passed but the normal run failed. Verbose output showed successful SSH authentication as student before the source lookup failed. That helped me separate a controller file problem from a connection problem.

**Observed after fixing that exercise:** serverb reported `ok=2 changed=1 failed=0`, including the completion message.

For extra detail:

```bash
ansible-navigator run troubleshoot.yml -i inventory-bab2-2 -m stdout -vvvv
```

Start with the failed task and its message. Add more verbosity when the message is not enough. Connection failures are usually reported as UNREACHABLE; module or task failures are normally FAILED. The actual error text points to what needs checking next.

The cryptography warnings appeared in many runs, including successful ones. They were separate from the YAML, file path and PHP problems fixed here.

## 15. Documentation, artifacts and execution controls

We can look up module options, required arguments and examples instead of remembering every parameter:

```bash
ansible-navigator doc ansible.builtin.file -m stdout
ansible-navigator doc ansible.builtin.systemd_service -m stdout
ansible-navigator doc ansible.posix.firewalld -m stdout
ansible-navigator doc ansible.builtin.uri -m stdout
```

These are reference commands. With the execution environment enabled, Navigator reads documentation from that environment. A direct `ansible-doc` command reads the modules available where that command runs, which can be different.

**Reference: saving a run for replay**

```bash
ansible-navigator run web-review.yml -i inventory-review -m stdout --pae true --pas web-review-artifact.json
ansible-navigator replay web-review-artifact.json -m stdout
```

`--pae` means `playbook-artifact-enable`. Its built in default is true, but the configuration recorded in the Chapter 1 notes disables it. It can be explicitly enabled for a run. `--pas` chooses the artifact filename. Use a different filename to keep another run separately.

Replay reads a saved execution record. It does not rerun the tasks or check the servers' current state. This behaviour was discussed during the chapter, but there is no verified replay transcript in these notes.

**Reference: inspecting or stepping through tasks**

```bash
ansible-navigator run web-review.yml -i inventory-review -m stdout --list-tasks
ansible-navigator run web-review.yml -i inventory-review -m stdout --step --pae false
ansible-navigator run web-review.yml -i inventory-review -m stdout --start-at-task "Start and enable httpd"
```

1. `--list-tasks` lists the tasks without executing them.
2. `--step` prompts before each task. Choose y to run it, n to skip it, or c to continue without more prompts. For this interactive input, use stdout mode and disable artifacts.
3. `--start-at-task` starts at the named task and continues afterwards. Earlier prerequisites must already exist.

I confused pae with "pause at each" at first. The pause control is step.

Adding `log_path = ./ansible.log` under the existing `[defaults]` section in `ansible.cfg` configures Ansible text logging. It is different from a Navigator JSON artifact and from `ansible-navigator.log`, which records Navigator diagnostics. A saved text log was not verified in the outputs recorded here.

## 16. Final review inventory and database playbook

The final `inventory-review`:

```ini
servera ansible_host=172.16.218.10

[web]
serverb ansible_host=172.16.218.11

[db]
serverc ansible_host=172.16.218.12
```

Servera is ungrouped, serverb is the web host and serverc is the database host. These final playbooks were run with `-i inventory-review`.

I repaired a database starter by adding a play name, using true for become, writing the dnf FQCN and capitalising a task name.

The corrected `database-review.yml`:

```yaml
---
- name: Set up the database server
  hosts: db
  become: true
  gather_facts: false
  tasks:
    - name: Install database packages
      ansible.builtin.dnf:
        name:
          - firewalld
          - mariadb-server
        state: present

    - name: Start and enable firewalld
      ansible.builtin.systemd_service:
        name: firewalld
        state: started
        enabled: true

    - name: Allow database traffic
      ansible.posix.firewalld:
        service: mysql
        permanent: true
        immediate: true
        state: enabled

    - name: Start and enable MariaDB
      ansible.builtin.systemd_service:
        name: mariadb
        state: started
        enabled: true
```

```bash
ansible-lint database-review.yml
ansible-navigator run database-review.yml -i inventory-review -m stdout --syntax-check
ansible-navigator run database-review.yml -i inventory-review -m stdout
```

**Observed:** Lint passed with zero failures and zero warnings, and syntax checking passed.

```text
First run:  serverc : ok=4 changed=3 unreachable=0 failed=0
Second run: serverc : ok=4 changed=0 unreachable=0 failed=0
```

Firewalld was already running and enabled, so its service task did not change anything on the first run. The other three tasks changed the server.

## 17. Final web playbook

The project file `files/index.php` contained:

```php
<?php
header('Content-Type: text/plain');
echo "RHCE Chapter 2 web test OK\n";
```

The corrected `web-review.yml`:

```yaml
---
- name: Setup web server review
  hosts: web
  become: true
  gather_facts: false
  tasks:
    - name: Install packages
      ansible.builtin.dnf:
        name:
          - httpd
          - firewalld
          - php
          - php-mysqlnd
          - php-fpm
        state: present

    - name: Start and enable firewalld
      ansible.builtin.systemd_service:
        name: firewalld
        state: started
        enabled: true

    - name: Allow firewall for http
      ansible.posix.firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled

    - name: Start and enable php-fpm
      ansible.builtin.systemd_service:
        name: php-fpm
        state: started
        enabled: true

    - name: Copy index file
      ansible.builtin.copy:
        src: files/index.php
        dest: /var/www/html/index.php
        owner: root
        group: root
        mode: '0644'

    - name: Start and enable httpd
      ansible.builtin.systemd_service:
        name: httpd
        state: started
        enabled: true

- name: Test the web server from servera
  hosts: servera
  become: false
  gather_facts: false
  tasks:
    - name: Check web status
      ansible.builtin.uri:
        url: http://172.16.218.11/index.php
        status_code: 200
        return_content: true
      register: web_result

    - name: Show web content
      ansible.builtin.debug:
        var: web_result.content
```

The HTTP request runs from servera to serverb. The URL uses `/index.php` explicitly because serverb still had an `index.html` page from an earlier exercise.

PHP uses PHP-FPM in this RHEL setup, so the playbook includes its package and service explicitly. Installing `php-mysqlnd` does not configure an application database connection by itself.

```bash
ansible-lint web-review.yml
ansible-navigator run web-review.yml -i inventory-review -m stdout --syntax-check
ansible-navigator run web-review.yml -i inventory-review -m stdout
```

**Observed:** Lint and syntax checks passed. The six setup tasks eventually reported no changes, but the content check exposed one more problem.

## 18. HTTP 200 did not mean PHP was working

The first useful debug output showed PHP source code:

```text
"web_result.content": "<?php\nheader('Content-Type: text/plain');\necho \"RHCE Chapter 2 web test OK\\n\";"
```

The request had succeeded, but Apache was serving the file without processing PHP. Apache had already been running from an earlier exercise, and the started task did not restart it to load the PHP configuration added by package installation.

I fixed this once on serverb:

```bash
ssh student@172.16.218.11 "sudo systemctl restart httpd"
ansible-navigator run web-review.yml -i inventory-review -m stdout
```

**Observed after the restart:**

```text
"web_result.content": "RHCE Chapter 2 web test OK\n"

servera : ok=2 changed=0 unreachable=0 failed=0 skipped=0
serverb : ok=6 changed=0 unreachable=0 failed=0 skipped=0
```

This verified the returned PHP output as well as the HTTP status. There were six setup tasks on serverb and two test tasks on servera, with no gathering facts task in either play.

The manual restart was outside the playbook, so it is not counted in that recap. Keeping `state: started` in the playbook avoids an unconditional restart on every run.

Repeating this exercise with Apache already running before PHP installation still needs a configuration reload. The Chapter 2 playbook does not automatically handle that transition. Service restarts will come up again when studying handlers.

## 19. Things to remember

1. Check the selected inventory before assuming which hosts a group contains.
2. YAML structure matters. Module parameters and task keywords are seperate things.
3. SSH login, privilege escalation and file ownership are different decisions.
4. A task reporting changed is reporting the module's result. Its meaning depends on what that module does.
5. Lint, syntax checking, check mode and normal execution answer different questions.
6. A successful recap is useful, but the actual service or content still needs checking.
7. Look up module options and examples instead of guessing parameter names.

I can now write and troubleshoot these small playbooks in my home lab. I still need practice writing them from requirements without looking back at my earlier files.

Reading references:

1. AU294 AAP 2.5, Chapter 2. The course material was the reference for these adapted home lab exercises.
2. [Ansible inventory guide](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html).
3. [Ansible URI module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html).
4. [Ansible Navigator settings](https://docs.ansible.com/projects/navigator/settings/).
5. [RHEL 10 programming languages guide, PHP with Apache](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/installing_and_using_dynamic_programming_languages/index).

[Back to index](README.md)
