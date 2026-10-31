# Chapter 3: Developing Automation Content: Variables

[Back to index](README.md)

Status: Completed my adapted Chapter 3 home lab exercises, including the final HTTPS authentication review in section 3.7. Section 3.8 is the chapter summary.

I used AU294 for AAP 2.5 on my RHEL 10 ARM lab. My setup is already in [the lab notes](00_lab_setup.md), and the earlier playbook basics are in [Chapter 2](02_chapter_two.md).

This chapter took me a few sessions. I sometimes referred to my earlier playbooks when I forgot a parameter. I still need practice writing these from requirements without looking back.

These notes follow my actual exercises. I cleaned up task names and made a few settings explicit in the examples. **Observed** means a result from my lab. **Reference** means a reminder or corrected example, not a claim that I ran every command shown. I did not run the Red Hat classroom grader. Password values and encrypted secret data are not included here.

For this public version, I replaced my homelab IP addresses with documentation-only addresses from 192.0.2.0/24 in both examples and displayed results. These are placeholders, not the addresses to use when running the commands. I need to substitute my own inventory values. The task counts, HTTP statuses and other reported results are unchanged.

## 1. What I worked on

1. Define variables inside a play and reuse them in tasks.
2. Move values into group_vars, host_vars and separate variable files.
3. Compare variable precedence.
4. Read dictionary keys, list items and registered task results.
5. Gather system facts, create custom facts and use magic variables.
6. Export facts from each host into JSON files.
7. Encrypt files and individual variables with Ansible Vault.
8. Configure an HTTPS server and test authentication using an encrypted password.

## 2. My Chapter 3 inventory

My `inventory-bab3` grouped the four managed hosts like this:

```ini
[prod]
servera ansible_host=192.0.2.10
serverb ansible_host=192.0.2.11

[dev]
serverc ansible_host=192.0.2.12
serverd ansible_host=192.0.2.13

[allservers:children]
prod
dev

[webserver]
serverb
```

I added `webserver` for the final review. Serverb still belongs to prod and allservers. Its existing address applies when it is selected through webserver too.

I used `-i inventory-bab3` for this chapter. When I forgot it in an ad hoc command, Ansible used a different inventory and reported that no hosts matched servera. The inventory option from an earlier command is not remembered by the next command.

## 3. Variables inside a play

My first exercise installed Cockpit and firewalld on allservers. I defined these play variables:

```yaml
vars:
  server_pkg: cockpit
  firewall_pkg: firewalld
  server_svc: cockpit.socket
  firewall_svc: firewalld
  firewall_rule: cockpit
```

I referenced them in tasks instead of repeating the values:

```yaml
- name: Install server and firewall packages
  ansible.builtin.dnf:
    name:
      - "{{ server_pkg }}"
      - "{{ firewall_pkg }}"
    state: present

- name: Start and enable Cockpit
  ansible.builtin.systemd_service:
    name: "{{ server_svc }}"
    state: started
    enabled: true
```

These are task fragments for a play's task list. I also started firewalld and enabled its cockpit service with both `permanent: true` and `immediate: true`.

I use `{{ variable_name }}` when inserting a value into a module parameter. When the whole YAML value starts with a Jinja expression, I quote it as shown above.

`vars` is a dictionary of names and values. `tasks` is a list of task dictionaries. I first forgot the list item under tasks in my user creation plays. This is the corrected shape:

```yaml
- name: Create the production user
  hosts: prod
  become: true
  vars:
    usr: alice
    uid: 8888
  tasks:
    - name: Ensure the user exists
      ansible.builtin.user:
        name: "{{ usr }}"
        uid: "{{ uid }}"
```

The development play used bob with UID 9999. I initially targeted prod by mistake, then corrected it to dev and removed the accidentally created bob account from prod.

**Observed:** After the correction, a second run of `variables-practice.yml` reported `changed=0` and `failed=0` on all four hosts.

## 4. Moving values into group_vars

I moved the shared service variables into `group_vars/allservers.yml`. The user settings became:

`group_vars/prod.yml`:

```yaml
---
usr: alice
uid: 8888
```

`group_vars/dev.yml`:

```yaml
---
usr: bob
uid: 9999
```

The file names match inventory group names. In my project, group_vars is beside the playbook and inventory.

I could then replace the two user plays with one:

```yaml
- name: Create the user selected by each host's group
  hosts: allservers
  become: true
  tasks:
    - name: Ensure the group user exists
      ansible.builtin.user:
        name: "{{ usr }}"
        uid: "{{ uid }}"
```

Even though the play targets allservers, variables are resolved for each host.

| Hosts | Group settings used | Account |
| --- | --- | --- |
| servera, serverb | prod | alice, UID 8888 |
| serverc, serverd | dev | bob, UID 9999 |

**Observed:** My combined service and user playbook completed with `ok=7 changed=0 failed=0` on each host.

One small formatting issue: `cat group_vars/*` joined the end of one file onto the start of the next because I had not saved a final newline. I now keep a newline at the end of each file.

## 5. Variable precedence I tested

I used `lab_message` to compare several places where a value can be defined.

| Test | Result |
| --- | --- |
| Group value only | Hosts displayed from_groups |
| Host value added for serverb | Serverb displayed from_host; the others still displayed from_groups |
| Play variable added | All four displayed from_play |
| Extra variable supplied with -e | All four displayed from_cli |

```bash
ansible-navigator run variables-precedence.yml -i inventory-bab3 -m stdout
ansible-navigator run variables-precedence.yml -i inventory-bab3 -m stdout -e lab_message=from_cli
```

For these sources, the order I observed from lower to higher priority was:

```text
group_vars < host_vars < play vars < extra vars (-e)
```

This is the comparison I practised, not the complete precedence list. I should not assume that a host variable overrides a play variable just because a host sounds more specific.

Extra vars override other variable definitions for that run. They do not rewrite my YAML files.

I also got a deprecation warning when I wrote vars as a list of dictionaries. The corrected form was:

```yaml
vars:
  lab_message: from_play
```

## 6. Variable files, dictionaries and lists

My `vars/lab.yml` contained:

```yaml
---
lab:
  course: RHCE
  platform: AAP 2.5
topics:
  - variables
  - inventory
report_path: /tmp/rhce-bab3-report.txt
```

I loaded it with `vars_files: vars/lab.yml`. A single filename is valid here. A YAML list of filenames can also be used.

My `variables-report.yml`:

```yaml
---
- name: Write my variable report
  hosts: allservers
  vars_files: vars/lab.yml
  tasks:
    - name: Write the report
      ansible.builtin.copy:
        content: |
          Course: {{ lab['course'] }}
          Platform: {{ lab['platform'] }}
          Topics:
            - {{ topics[0] }}
            - {{ topics[1] }}
        dest: "{{ report_path }}"
        mode: '0644'
      register: report_result

    - name: Inspect the copy result
      ansible.builtin.debug:
        var: report_result
```

`lab` is a dictionary, so I select keys such as `lab['course']`. `topics` is a list, so I select positions starting at zero.

I first wrote `topics[0],[1]`, which did not select two items from topics as I intended. I corrected it to two separate expressions. I also removed quotes from inside the `content: |` block because those quotes were being written into the file.

**Observed content:**

```text
Course: RHCE
Platform: AAP 2.5
Topics:
  - variables
  - inventory
```

After updating the content, copy reported changed. The next run reported `changed=0` on all four hosts.

## 7. What register actually stores

`register: report_result` saves the task's returned result for that host. It does not automatically save every input parameter or the destination file's content.

I tried `report_result.content` after a copy task and got `VARIABLE IS NOT DEFINED!`. Inspecting the whole result showed fields such as:

```text
changed: false
dest: /tmp/rhce-bab3-report.txt
mode: 0644
size: 67
```

The copy result did not contain a content field. Later, the uri module did return `auth_test.content` because I requested `return_content: true`.

| Result | What I can use it for |
| --- | --- |
| report_result.changed | Whether the copy task reported a change |
| report_result.dest | The returned destination path |
| auth_test.content | The response body returned by my URI request |
| auth_test.status | The HTTP response status returned by URI |

The available fields depend on the module and the result. I inspect the result or the module's Return Values documentation before guessing a field name.

With `debug.var`, I provide the variable expression directly:

```yaml
ansible.builtin.debug:
  var: report_result.changed
```

With `debug.msg`, I use Jinja when inserting a value into a message:

```yaml
ansible.builtin.debug:
  msg: "Report changed: {{ report_result.changed }}"
```

## 8. Facts in my host report

I extended the report with information collected from each server:

```yaml
content: |
  Inventory name: {{ inventory_hostname }}
  Hostname: {{ ansible_facts['hostname'] }}
  OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
  Architecture: {{ ansible_facts['architecture'] }}
  IPv4: {{ ansible_facts['default_ipv4']['address'] }}
```

This fragment goes under copy, with the destination and mode from the report task.

**Observed on servera:**

```text
Inventory name: servera
Hostname: servera
OS: RedHat 10.0
Architecture: aarch64
IPv4: 192.0.2.10
```

I first used the list of IPv4 addresses, which printed `['192.0.2.10']`. To get the address associated with the default IPv4 interface, I needed the address key inside the default_ipv4 dictionary.

```text
ansible_facts -> default_ipv4 -> address
```

That path is a nested dictionary lookup. It is different from selecting `topics[0]` from a list. The default IPv4 fact depends on the host having the relevant network information available.

Automatic fact gathering runs at the start of a play unless disabled. I can inspect the collected data with:

```yaml
- name: Inspect collected facts
  ansible.builtin.debug:
    var: ansible_facts
```

`inventory_hostname` comes from inventory. The hostname and FQDN facts come from the managed machine. They happen to be related in my lab, but they are not the same source.

## 9. Custom facts and set_fact

On servera, I created `/etc/ansible/facts.d` with mode 0755, then wrote `custom.fact` with mode 0644:

```ini
[general]
package = cockpit
service = cockpit.socket
state = started
enabled = true
```

My play used `gather_facts: false`. After creating the file, I explicitly ran setup so the new local facts were available in the same play:

```yaml
- name: Gather facts after creating the custom fact file
  ansible.builtin.setup:

- name: Inspect local facts
  ansible.builtin.debug:
    var: ansible_facts['ansible_local']

- name: Keep the custom settings in a shorter variable
  ansible.builtin.set_fact:
    custom: "{{ ansible_facts['ansible_local']['custom']['general'] }}"

- name: Install the requested package
  ansible.builtin.dnf:
    name: "{{ custom['package'] }}"
    state: present

- name: Configure the requested service
  ansible.builtin.systemd_service:
    name: "{{ custom['service'] }}"
    state: "{{ custom['state'] }}"
    enabled: "{{ custom['enabled'] }}"
```

The lookup contains the file basename `custom` and the INI section `general`. The returned enabled value was the string `"true"`; the module accepted it for its boolean parameter in my environment.

The `.fact` file persists on servera. My set_fact task creates a host variable during execution; it does not write that file or make the variable survive a later run by itself.

I initially wrote a literal word as the set_fact value instead of referencing the collected dictionary. I also used `custom['package']` as the service name. That managed cockpit instead of the requested cockpit.socket. Lint and a repeated run did not catch that mismatch with my intended service. I corrected it to `custom['service']`.

## 10. Magic variables and another host's facts

I used these expressions in `facts-inventory.yml`:

| Expression | Meaning in my exercise |
| --- | --- |
| inventory_hostname | The current inventory host |
| group_names | Groups containing the current host |
| groups['prod'] | Inventory names of hosts in prod |
| hostvars['serverb']['ansible_facts']['default_ipv4']['address'] | Serverb's gathered default IPv4 address |
| playbook_dir | The playbook directory on the controller side |

**Observed:** Servera and serverb showed allservers and prod. Serverc and serverd showed allservers and dev. Every host displayed serverb's IP as 192.0.2.11 when I used hostvars.

My play gathered facts for allservers before running the debug task. Looking up another host in hostvars does not gather its facts automatically. Those facts must already be available from gathering or an appropriate fact cache.

## 11. Exporting facts to JSON

My corrected `facts-export.yml`:

```yaml
---
- name: Export facts from each server to JSON
  hosts: allservers
  gather_facts: false
  become: false
  tasks:
    - name: Collect hardware and minimum facts
      ansible.builtin.setup:
        gather_subset:
          - hardware

    - name: Create each host's output directory
      ansible.builtin.file:
        path: "{{ playbook_dir }}/facts-output/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: Write facts as JSON
      ansible.builtin.copy:
        content: "{{ ansible_facts | to_nice_json }}"
        dest: "{{ playbook_dir }}/facts-output/{{ inventory_hostname }}/facts.json"
        mode: '0644'
      delegate_to: localhost
```

I first tried debug for directory creation. Debug only displays information; the file module creates the directory.

Setup runs on each managed server. The two delegated tasks run on localhost. With my execution environment enabled, that means the execution environment's local context. I used the mounted project path so I could keep the output in my project.

The per-host directory uses the original `inventory_hostname`, so each host gets its own facts.json. `to_nice_json` formats the dictionary as readable JSON.

**Observed creation run:** Each of the four hosts reported `ok=3 changed=2 failed=0`. The output showed `servera -> localhost` and the equivalent for the other hosts.

Hardware is not a synonym for only minimum facts. This subset selection also retains the minimum facts. Some collected values can change between runs, so a later JSON snapshot can legitimately change too.

## 12. My first Vault exercise

I used an encrypted `vars/vault.yml` containing app_user and app_password. My play loaded it with vars_files and wrote `/etc/rhce-vault.conf` on prod:

```yaml
- name: Write the application configuration
  ansible.builtin.copy:
    content: |
      user= {{ app_user }}
      password= {{ app_password }}
    dest: /etc/rhce-vault.conf
    owner: root
    group: root
    mode: '0600'
  no_log: true
```

I initially misspelled the output field password. A successful task can still write the wrong text when I make a mistake in content.

```bash
ansible-navigator run vault-practice.yml -i inventory-bab3 -m stdout --pae false --ask-vault-pass
```

**Observed:** The first successful run changed one task on each prod host. The second reported `changed=0` on both.

I need to keep three things seperate:

| Item | Purpose |
| --- | --- |
| Application password | Data the application or web login uses |
| Vault password | Opens the encrypted source data for Ansible |
| Vault ID label | Identifies a password source, such as prod, dev or web-user |

Vault encryption protects the source data at rest. The generated configuration on the managed host contains the usable plaintext values. File permissions and task logging still matter.

`no_log: true` suppresses that task's normal output. It does not encrypt the destination file and does not protect a later, separate debug or cat command.

## 13. Vault commands and password files

These are the commands I want available as a reference. The authentication options depend on which Vault sources I have configured for that exercise.

| Command | Purpose |
| --- | --- |
| ansible-vault create vars/vault.yml | Create a new encrypted file through an editor |
| ansible-vault view vars/vault.yml | View decrypted content without leaving the source decrypted |
| ansible-vault edit vars/vault.yml | Edit content and save it encrypted |
| ansible-vault encrypt vars/secret.yml | Encrypt an existing plaintext file |
| ansible-vault decrypt vars/secret.yml | Leave the file decrypted on disk |
| ansible-vault rekey vars/vault.yml | Change the encryption password for an encrypted file |

Decrypting needs the password that encrypted the data. Rekeying does not change the application values inside the file. For an individually encrypted variable, I generate a replacement encrypted value instead of using file rekey on the surrounding mixed YAML file.

I first practised automatic password loading with a `.vault-pass` file and `vault_password_file` in ansible.cfg. Later, the final review used a named identity and a different password file.

**Final review configuration**, under the existing defaults section:

```ini
[defaults]
vault_identity_list = web-user@.web_pass
```

The `.web_pass` file contains the Vault password on one line. It is a plaintext password file, even though its name begins with a dot.

```bash
chmod 600 .web_pass
```

I created `.gitignore` in my project folder and added the password file names:

```gitignore
.web_pass
.vault-pass
```

This keeps new, untracked password files out of Git. It does not remove a file that has already been committed. I do not store the actual password files or their contents in this notes repository.

## 14. Vault IDs and the CLI mistakes I made

For prod and dev, I used separate encrypted variable files and two plays. Both files used the same variable names, app_user and app_password, but each play loaded the file for its environment.

```bash
ansible-navigator run vault-environments.yml -i inventory-bab3 -m stdout --pae false --vault-id prod@prompt --vault-id dev@prompt
```

**Observed:** I got a prompt for each identity. The headers ended in prod and dev, and the second run reported no changes on all four hosts.

The syntax for a labelled password source is:

```text
--vault-id label@source
```

Examples are `prod@prompt` or `web-user@.web_pass`. The label is not the password itself, and it does not select the target hosts. Hosts are still selected by the play and inventory.

I made two useful mistakes in the final review:

| What I entered or saw | What happened |
| --- | --- |
| --vault-id web-user | Ansible looked for a password file named web-user and warned that it did not exist |
| Encryption successful after that warning | Encryption still completed using an available password source; I needed to inspect the result rather than assume it failed |
| --vault-id web-user@.web_pass with the same identity already in config | My environment reported two available entries: web-user,web-user |
| Command without the repeated CLI identity | The intended way to use the single identity already configured in ansible.cfg |

With only the final review identity configured, the encryption command is simply:

```bash
ansible-vault encrypt vars/secret.yml
```

This command is for a plaintext file. I should not encrypt it again just because I saw a warning in an earlier command.

The error for multiple identities mentioned `--encrypt-vault-id`. That option selects an encryption label when multiple password sources are intentionally available. For my single-identity setup, removing the duplicate CLI source was simpler.

I can inspect the header without displaying the secret values:

```bash
head -1 vars/secret.yml
```

Expected for the labelled final review file:

```text
$ANSIBLE_VAULT;1.2;AES256;web-user
```

One earlier encrypted file had a valid header without a label:

```text
$ANSIBLE_VAULT;1.1;AES256
```

It was still encrypted and usable with the correct password. I later rekeyed it to add lab as the label. Passing a labelled password source during a playbook run does not rewrite the file's header.

By default, labels help Ansible choose which supplied password to try first. A label is not an extra security boundary. The actual password must decrypt the data.

## 15. Encrypting one variable and copying an encrypted file

I practised a mixed variable file: app_user stayed readable, while app_password used an encrypted value.

**Reference command for an exercise using the lab identity, without conflicting default Vault sources:**

```bash
ansible-vault encrypt_string --vault-id lab@prompt --stdin-name app_password
```

After entering the Vault password, I supply the string to encrypt through standard input and finish with EOF. I avoid adding an unintended trailing newline to the secret. In my interactive terminal, Ctrl+D twice after the last character finished the input.

I originally forgot the two hyphens before stdin-name. Ansible then encrypted the literal command arguments stdin-name and app_password and produced two unnamed blocks. I discarded those and reran the corrected command.

The resulting block belongs under `app_password: !vault |`. I keep its indentation and copy the whole encrypted block. I do not copy a prompt or success message into the YAML file.

My mixed exercise also copied a completely encrypted source file:

```yaml
- name: Copy the encrypted application file
  ansible.builtin.copy:
    src: files/vault-app.conf
    dest: /etc/rhce-file-vault.conf
    owner: root
    group: root
    mode: '0600'
    decrypt: true
  no_log: true
```

The normal controller-side copy action can decrypt the source when Ansible has the correct Vault password. The managed destination receives plaintext. `decrypt: true` is already the copy module's default; setting it explicitly makes the intention visible. `decrypt: false` would copy the encrypted bytes instead.

**Observed:** My two-task `vault-mixed.yml` changed twice on the first successful run and reported `changed=0` on the next. After successfully rekeying the full source file with the lab label, the destination still needed no change because its plaintext content stayed the same.

## 16. Final review: HTTPS with authentication

The final exercise combined variables, facts, Vault and registered output. Serverb was the web server. Servera made the HTTPS request.

The course environment normally provides starter files. For my homelab, I prepared `files/bab3-review/httpd.conf`, `.htaccess` and `htpasswd` from my existing Apache setup.

Inside the `<Directory "/var/www/html">` block of the local httpd.conf copy, I allowed the authentication directives with:

```apache
AllowOverride AuthConfig
```

My `.htaccess` content was:

```apache
AuthType Basic
AuthName "Bab 3 Review"
AuthUserFile /etc/httpd/secrets/htpasswd
Require valid-user
```

The htpasswd file contained a hash for the guest HTTP login. Guest is a web authentication user here; this did not require creating a Linux account named guest.

I stored the matching web_pass variable in `vars/secret.yml`, encrypted that file with the web-user identity, and used the `.web_pass` configuration shown earlier. The web login password and the Vault password serve different purposes.

My final `webserver-review.yml`, with task names cleaned up and the expected HTTP status written explicitly:

```yaml
---
- name: Configure the review web server
  hosts: webserver
  become: true
  gather_facts: true
  vars:
    firewall_pkg: firewalld
    firewall_svc: firewalld
    web_pkg: httpd
    web_svc: httpd
    ssl_pkg: mod_ssl
    httpdconf_src: files/bab3-review/httpd.conf
    httpdconf_dest: /etc/httpd/conf/httpd.conf
    htaccess_src: files/bab3-review/.htaccess
    secrets_dir: /etc/httpd/secrets
    secrets_src: files/bab3-review/htpasswd
    secrets_dest: "{{ secrets_dir }}/htpasswd"
    web_root: /var/www/html
  tasks:
    - name: Install the required packages
      ansible.builtin.dnf:
        name:
          - "{{ firewall_pkg }}"
          - "{{ web_pkg }}"
          - "{{ ssl_pkg }}"
        state: present

    - name: Copy the Apache configuration
      ansible.builtin.copy:
        src: "{{ httpdconf_src }}"
        dest: "{{ httpdconf_dest }}"
        owner: root
        group: root
        mode: '0644'

    - name: Create the authentication directory
      ansible.builtin.file:
        path: "{{ secrets_dir }}"
        state: directory
        owner: apache
        group: apache
        mode: '0500'

    - name: Copy the password hash file
      ansible.builtin.copy:
        src: "{{ secrets_src }}"
        dest: "{{ secrets_dest }}"
        owner: apache
        group: apache
        mode: '0400'

    - name: Copy the directory authentication rules
      ansible.builtin.copy:
        src: "{{ htaccess_src }}"
        dest: "{{ web_root }}/.htaccess"
        owner: apache
        group: apache
        mode: '0400'

    - name: Write a page using server facts
      ansible.builtin.copy:
        content: |
          {{ ansible_facts['fqdn'] }} ({{ ansible_facts['default_ipv4']['address'] }}) has been customized by Ansible.
        dest: "{{ web_root }}/index.html"
        mode: '0644'

    - name: Start and enable firewalld
      ansible.builtin.systemd_service:
        name: "{{ firewall_svc }}"
        state: started
        enabled: true

    - name: Allow HTTPS now and permanently
      ansible.posix.firewalld:
        service: https
        permanent: true
        immediate: true
        state: enabled

    - name: Start and enable Apache
      ansible.builtin.systemd_service:
        name: "{{ web_svc }}"
        state: started
        enabled: true

- name: Test the web server from servera
  hosts: servera
  become: false
  vars:
    web_user: guest
  vars_files: vars/secret.yml
  tasks:
    - name: Request the page using basic authentication
      ansible.builtin.uri:
        url: https://192.0.2.11/
        user: "{{ web_user }}"
        password: "{{ web_pass }}"
        force_basic_auth: true
        validate_certs: false
        status_code: 200
        return_content: true
      no_log: true
      register: auth_test

    - name: Display the returned page
      ansible.builtin.debug:
        var: auth_test.content
```

This playbook depends on the prepared local configuration and encrypted variable files. The notes do not include the actual htpasswd data or Vault password file.

I used an IP in the URL because Ansible inventory aliases do not automatically become DNS names available to servera. The FQDN in the page comes from serverb's facts.

The URI module accepts user and password as aliases for url_username and url_password. Its default successful status is already 200. I wrote status_code explicitly in this notes version to make the test clearer. Certificate validation is disabled for this controlled lab's self-signed certificate.

I first debugged the entire auth_test result. I changed it to auth_test.content so the task displays the page requested by the exercise. The no_log setting on the URI task does not carry over to another debug task.

One remaining limitation is configuration reloads. `state: started` does not restart an already running Apache process when httpd.conf changes. When repeating this setup on the server left over from Chapter 2, I need to account for a one-time service restart after such changes. This playbook does not automate that transition with a handler. An unconditional restarted task would restart Apache on every run, so I kept the chapter's started behaviour.

## 17. My final verification

```bash
ansible-navigator run webserver-review.yml -i inventory-bab3 -m stdout --pae false
```

**Observed returned content:**

```text
serverb.lab.example.com (192.0.2.11) has been customized by Ansible.
```

**Observed repeated-run recap:**

```text
servera : ok=3  changed=0 unreachable=0 failed=0 skipped=0
serverb : ok=10 changed=0 unreachable=0 failed=0 skipped=0
```

There were nine setup tasks plus fact gathering on serverb. Servera had fact gathering, the URI request and the debug task. The URI request passed its expected HTTP 200 check and returned the correct serverb content.

I also tested the same URL from servera without sending login credentials:

```bash
ssh student@192.0.2.10 'curl -ks -o /dev/null -w "%{http_code}\n" https://192.0.2.11/'
```

**Observed:**

```text
401
```

My first attempt had a missing quote, then I typed `%http_code}` without the opening brace. Curl printed that text instead of a status code. After correcting it to `%{http_code}`, I got the 401 result.

| Check | Result |
| --- | --- |
| Authenticated request from servera | Passed the URI HTTP 200 check |
| Returned content | Correct serverb FQDN and IP |
| Request without login | HTTP 401 |
| Repeated playbook run | No changes and no failures |

The 401 check matters because getting a page with credentials alone would not prove that the server rejects requests without credentials.

## 18. What I want to remember from section 3.8

1. Variables store values that my tasks reuse. I need to know which source supplies the winning value.
2. Dictionary keys and list positions are different ways of selecting data.
3. Register saves a task result. I should inspect the returned fields instead of assuming every result has content.
4. Facts describe managed hosts. Custom fact files add data that setup can collect.
5. Magic variables describe inventory and execution context. Hostvars can expose another host's facts when those facts are available.
6. Vault protects sensitive source data at rest. Its password, its label and the application password are different things.
7. A clean recap is useful, but I still verify the content and behaviour the exercise asked for.

I finished the adapted chapter, but I do not yet remeber every option without checking. My next revision should be one combined exercise from a blank playbook, then I can compare it with these notes.

My reading references:

1. AU294 AAP 2.5, Chapter 3, including the final review and summary. These notes use my own explanations and homelab results.
2. [Ansible variables](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html).
3. [Facts and magic variables](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_vars_facts.html).
4. [Encrypting content with Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_encrypting_content.html).
5. [Managing Vault passwords](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_managing_passwords.html).
6. [URI module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html).
7. [Copy module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/copy_module.html).
8. [Ansible output logging and no_log](https://docs.ansible.com/projects/ansible/latest/reference_appendices/logging.html).
9. [Apache authentication](https://httpd.apache.org/docs/2.4/howto/auth.html).

[Back to index](README.md)
