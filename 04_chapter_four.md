# Chapter 4: Developing Automation Content: Task Control

[Back to index](README.md)

Status: Home lab practice completed for loops, conditions, handlers and error handling, including a combined exercise tested through recovery and a repeated run with no changes.

These notes use AU294 for AAP 2.5 with the RHEL 10 ARM home lab. The earlier setup is in [the lab notes](00_lab_setup.md), and variables and registered results are in [Chapter 3](03_chapter_three.md).

I understand the flow better after running the exercises, but I still need to check notes when writing a combined playbook. This chapter needs another revision later.

The examples below follow our practice. Task names have been cleaned up where needed. **Observed** means a result recorded during the lab. **Reference** means an explanation or example, not a claim that it was tested. These were adapted home lab exercises, not a run of the official classroom grader or a claim that every guided exercise was completed.

## 1. Chapter map and lab commands

The book groups this chapter into three main topics:

| Section | Topic |
| --- | --- |
| 4.1 and 4.2 | Loops and conditional tasks, followed by a guided exercise |
| 4.3 and 4.4 | Handlers, followed by a guided exercise |
| 4.5 and 4.6 | Task failure, followed by a guided exercise |
| 4.7 and 4.8 | Combined chapter lab and summary |

The practice in this session targeted `servera` using the existing `inventory-bab2-2` inventory. Servera is the same home lab host at `172.16.218.10`.

```bash
ansible-navigator run bab4-practice.yml -i inventory-bab2-2 -m stdout
ansible-navigator run bab4-handler.yml -i inventory-bab2-2 -m stdout
ansible-navigator run bab4-error.yml -i inventory-bab2-2 -m stdout
ansible-navigator run bab4-combined.yml -i inventory-bab2-2 -m stdout
```

Syntax checking uses the same inventory and execution environment:

```bash
ansible-navigator run bab4-combined.yml -i inventory-bab2-2 -m stdout --syntax-check
```

The playbooks below keep the privilege settings used in the recorded attempts. The existing lab connection and privilege escalation configuration applies. On a fresh setup, configure the required access before trying to change files owned by another user.

## 2. Looping over directories

`loop` runs a task once for each list entry. By default, `item` holds the current entry.

The corrected `bab4-practice.yml`:

```yaml
---
- name: Practice Chapter 4 loops
  hosts: servera
  vars:
    practice_directories:
      - app1
      - app2
      - app3
      - app4
  tasks:
    - name: Ensure the directories exist
      ansible.builtin.file:
        path: "/tmp/rhce/{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ practice_directories }}"
      register: directory_results

    - name: Display changed directories
      ansible.builtin.debug:
        msg: "Directory {{ item['path'] }} was created or updated"
      loop: "{{ directory_results.results }}"
      when: item['changed']
```

In the first task, `item` is a string such as `app1`. In the second task, `item` is a complete result dictionary from one iteration of the first task.

**Observed:** App1, app2 and app3 already existed. Adding app4 produced this behaviour:

| Directory | File task | Debug task |
| --- | --- | --- |
| app1, app2, app3 | ok | Skipped |
| app4 | changed | Displayed its path |

The recap was `ok=3 changed=1 failed=0 skipped=0`. Some loop items were skipped, but the debug task ran for app4. Recap counters are task-level totals for the host, not a count of every skipped loop item.

## 3. Understanding directory_results.results

With a loop, the registered variable contains a `results` list. This is a simplified shape, not the full module output:

```yaml
directory_results:
  changed: true
  results:
    - item: app1
      path: /tmp/rhce/app1
      changed: false
    - item: app4
      path: /tmp/rhce/app4
      changed: true
```

`directory_results.changed` describes whether the looping task reported a change overall. The individual entries let us check which directory changed.

Inside the second loop:

| Expression | Meaning |
| --- | --- |
| item | The current result dictionary |
| item.item or item['item'] | The original loop value, such as app4 |
| item.path or item['path'] | The path returned by the file module |
| item.changed or item['changed'] | Whether that iteration changed anything |

`item.item` is not a special double-loop command. The first `item` is the current variable; the second selects a field inside it.

## 4. The boolean mistake

My first condition was:

```yaml
when: item['changed'] == "true"
```

The quoted value is a string. The result's `changed` field is a boolean, so this comparison did not select changed items correctly.

The correction was:

```yaml
when: item['changed']
```

To select unchanged items instead:

```yaml
when: not item['changed']
```

A run where everything is already unchanged does not prove the first condition is correct. Adding app4 gave us a changed item and unchanged items in the same run, which tested both outcomes.

`when` is evaluated for each loop item. It does not need `{{ }}` around the expression.

## 5. Other conditions to recognise

Reference fragments for a task looping over directory results:

```yaml
when:
  - item.changed
  - item.item == "app1"
```

A list of conditions means all must be true. The equivalent is:

```yaml
when: item.changed and item.item == "app1"
```

Use `or` when either condition is enough:

```yaml
when: item.item == "app1" or item.item == "app2"
```

To check whether a variable exists:

```yaml
when: directory_results is defined
```

These are reference expressions. The recorded directory practice used the simple `when: item['changed']` condition.

## 6. Handlers and notify

A handler is a task notified by another task when that task reports `changed`. A common use is restarting a service after its configuration changes.

The practice used debug messages so the order was easy to see. `/tmp/rhce` was already created by the earlier directory exercise.

```yaml
---
- name: Practice handlers
  hosts: servera
  tasks:
    - name: Write the configuration
      ansible.builtin.copy:
        content: 'version=2'
        mode: '0644'
        dest: /tmp/rhce/practice.conf
      notify: Report config change

    - name: Display the normal task message
      ansible.builtin.debug:
        msg: Normal task completed.

  handlers:
    - name: Report config change
      ansible.builtin.debug:
        msg: Configuration changed.
```

**Observed after changing the content:** Copy reported changed, the normal debug task ran, and then the handler ran. The recap was `ok=4 changed=1 failed=0`.

**Observed on the next run:** Copy reported ok, the normal debug still ran, and the handler did not run. The recap was `ok=3 changed=0 failed=0`.

The file existing is not enough to decide whether the handler runs. Different content, or a requested file attribute that needs changing, can make copy report changed.

For this simple play, handlers run after the normal tasks. They are not executed immediately when `notify` is encountered. Multiple notifications of the same handler before a handler flush produce one execution per host. Different handlers run in their definition order, not notification order.

Reference task for running pending handlers earlier:

```yaml
- name: Run pending handlers now
  ansible.builtin.meta: flush_handlers
```

This was discussed, but not used in the recorded practice. Handler execution also has boundaries around pre_tasks, tasks/roles and post_tasks in more complex plays.

## 7. Choosing when or a handler

| Requirement | Approach |
| --- | --- |
| Display each changed directory's result | Loop over registered results and use when |
| Restart a service after changing its config | Notify a handler |
| Display a message immediately if a single task changed | Register the task, then use when on its changed field |
| Group tasks and recover from a failure | Use block and rescue |

A handler can also use conditions or loops when needed. The distinction here is the behaviour required by the exercise: inspect each result now, or schedule an action after a change.

## 8. Task failure and changed status

By default, a failed task stops further normal tasks on that host. Other hosts can continue.

| Keyword | Purpose |
| --- | --- |
| ignore_errors | Continue after a normal task failure |
| failed_when | Define when the task result counts as failure |
| changed_when | Define when the task reports a change |
| force_handlers | Run already notified handlers despite a later task failure |

`changed_when: false` does not skip a command or stop it changing the server. It controls the reported changed status and therefore affects notifications. A real failure can still be reported as failed.

Reference example for a command that only reads information:

```yaml
- name: Read uptime
  ansible.builtin.command: uptime
  changed_when: false
```

`ignore_errors: true` was discussed as a reference. It does not handle every error type, such as an unreachable host or an invalid task definition. An ignored task failure does not trigger rescue.

## 9. Block, rescue and always

The first error-handling exercise ran these steps inside a block:

1. Display `Starting application check`.
2. Run `/bin/false`.
3. Display `Application check passed`.

`/bin/false` returns exit code 1. It does not modify the server. With the default failure handling, the command failed and the third task was not reached.

The rescue section displayed the failed task's name. Always displayed `Check completed`, and a normal task after the block displayed `Play continues`.

**Observed:** The output included `Failed task: Force failure`, and the recap showed `failed=0 rescued=1`.

| Part | When it runs |
| --- | --- |
| block | The main tasks |
| rescue | After a task in the block returns failed |
| always | After block/rescue, regardless of their normal success or failure |

We can use block alone, block with rescue, block with always, or all three. Rescue and always belong to a block. Invalid task definitions and unreachable hosts are exceptions to normal block error handling.

Successful rescue handles the failure so execution can continue. It does not automatically undo changes or retry the failed task. Recovery only does what its tasks explicitly request.

Ansible supplies these variables inside rescue without requiring a separate register:

| Variable | Contains |
| --- | --- |
| ansible_failed_task | The task that failed and triggered rescue |
| ansible_failed_task.name | That task's name |
| ansible_failed_result | Its returned result |

```yaml
rescue:
  - name: Report the failed task
    ansible.builtin.debug:
      msg: "Failed task: {{ ansible_failed_task.name }}"
```

## 10. Changing the failure condition

The next attempt kept `/bin/false` but changed the task to:

```yaml
- name: Test a custom failure condition
  ansible.builtin.command: /bin/false
  register: command_result
  failed_when: command_result.rc > 1
  changed_when: false
```

The command still returned 1. The new condition only counts values greater than 1 as failure, so Ansible accepted the result.

**Observed:** The command reported ok, `Application check passed` was displayed, rescue did not run, and the recap was `ok=6 changed=0 failed=0 rescued=0`.

This deliberately accepts return code 1 to demonstrate the condition. It is not a general rule for application checks; choose accepted return codes according to the command being used.

The important separation is between the command's result and Ansible's interpretation of that result.

## 11. Explicit failure and force_handlers

The handler exercise was extended with a failure immediately after copy:

```yaml
- name: Simulate failure
  ansible.builtin.fail:
    msg: Simulated failure
```

Unlike failed_when on another module, fail is a task whose purpose is to report failure. It can be combined with when to reject an unsupported condition.

Without force_handlers, the changed copy task notified the handler, but the later failure stopped the normal debug task and prevented the pending handler from running on servera.

Then the play used:

```yaml
- name: Practice handlers after failure
  hosts: servera
  force_handlers: true
  tasks:
    # Copy, explicit failure and normal debug tasks go here.
```

This is a placement fragment. The full play still needs its tasks and handlers. `force_handlers` is at play level, alongside hosts and tasks. It can also appear below handlers at the same indentation; key order is not the issue.

**Observed comparison:**

| Setting | Normal debug after failure | Pending handler | Recap |
| --- | --- | --- | --- |
| Without force_handlers | Not run | Not run | ok=2 changed=1 failed=1 |
| With force_handlers: true | Not run | Run | ok=3 changed=1 failed=1 |

The copy content was changed between tests to generate a new notification. Reusing identical content would report ok and would not notify anything.

Force_handlers does not create notifications or ignore failure. It runs handlers already notified; an unreachable host can still prevent execution.

## 12. The combined exercise mistake

The first combined attempt registered a single copy task as `file_results`, then tried:

```yaml
loop: "{{ file_results.results }}"
```

**Observed error:**

```text
'dict object' has no attribute 'results'
```

That copy task did not have a loop, so its result did not have a loop results list. A direct changed check would use `file_results.changed`.

For this exercise, the requirement was to schedule a message after a configuration change, so the correction was to use notify and a handler instead. The unnecessary register and debug loop were removed.

Two other corrections were needed: add the missing when to the directory messages, and add changed_when: false to the file existence check. The block structure itself was already correct.

## 13. Final combined playbook

The final `bab4-combined.yml`, with descriptive task names:

```yaml
---
- name: Combined Chapter 4 practice
  hosts: servera
  vars:
    combined_directories:
      - app1
      - app2
      - app3
  tasks:
    - name: Create directories
      ansible.builtin.file:
        path: "/tmp/rhce-combined/{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ combined_directories }}"
      register: directory_results

    - name: Display changes only
      ansible.builtin.debug:
        msg: "{{ item['path'] }}"
      loop: "{{ directory_results.results }}"
      when: item['changed']

    - name: Write the application configuration
      ansible.builtin.copy:
        content: 'version=3'
        dest: /tmp/rhce-combined/app.conf
        mode: '0644'
      notify: Report config change

    - name: Validate application readiness
      block:
        - name: Starting validation
          ansible.builtin.debug:
            msg: Starting validation

        - name: Checking file
          ansible.builtin.command: /usr/bin/test -f /tmp/rhce-combined/ready
          changed_when: false

        - name: Report successful validation
          ansible.builtin.debug:
            msg: Application is ready

      rescue:
        - name: Show failed task
          ansible.builtin.debug:
            msg: "{{ ansible_failed_task.name }}"

        - name: Create readiness file
          ansible.builtin.copy:
            dest: /tmp/rhce-combined/ready
            content: ""
            mode: '0644'

      always:
        - name: Report validation completion
          ansible.builtin.debug:
            msg: Validation attempt completed

    - name: Report continued execution
      ansible.builtin.debug:
        msg: Play continues

  handlers:
    - name: Report config change
      ansible.builtin.debug:
        msg: Application configuration changed
```

The readiness file is a simple lab marker. Its existence does not prove that a real application is healthy. The test returns nonzero when the path is missing or is not a regular file. In our recorded run, it was missing.

The file module creates the directory path, including missing parent directories. The following copy task therefore has an existing destination parent.

## 14. Recorded verification

The directories already existed from earlier attempts before the final recovery run. This was not a completely fresh first run.

| Step | Recovery run | Immediate repeated run |
| --- | --- | --- |
| Directories | Already correct | Already correct |
| Directory messages | Skipped | Skipped |
| Configuration | Changed to version=3 | No change |
| Readiness check | Failed because ready was missing | Passed |
| Application is ready | Not reached | Displayed |
| Rescue | Reported Checking file and created ready | Not run |
| Always | Displayed completion | Displayed completion |
| Task after block | Displayed Play continues | Displayed Play continues |
| Handler | Displayed config change at the end | Not run |

**Observed recovery recap:**

```text
servera : ok=9 changed=2 unreachable=0 failed=0 skipped=1 rescued=1 ignored=0
```

The two changes were the configuration update and creation of ready. The checking command reported changed=false even when it failed.

No force_handlers was needed in this combined play. Rescue succeeded, so the host continued and the previously notified handler ran normally.

**Observed repeated-run recap:**

```text
servera : ok=8 changed=0 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
```

This tested both the recovery path and the successful repeated run. Rescue did not rerun the check in the recovery run; the next playbook run checked the new file successfully.

## 15. Revision reminders

| If the requirement says... | Think about... |
| --- | --- |
| Do the same task for a list | loop |
| Save a task's returned information | register |
| Read every result from a looping task | registered_variable.results |
| Run a task only when a condition is true | when |
| Run an action after another task changes something | notify and handlers |
| Continue after an expected task failure | ignore_errors, when appropriate |
| Decide what counts as failure | failed_when |
| A checking command should not report a change | changed_when: false |
| Explicitly reject a condition with a message | ansible.builtin.fail with when |
| Recover after a group of tasks fails | block and rescue |
| Perform a final action after normal success or failure | always |
| Run pending handlers despite a later failure | force_handlers |

Useful module documentation commands:

```bash
ansible-doc ansible.builtin.file
ansible-doc ansible.builtin.copy
ansible-doc ansible.builtin.debug
ansible-doc ansible.builtin.command
ansible-doc ansible.builtin.fail
```

For playbook structure such as blocks and handlers, use the playbook guide references below. Module documentation and playbook keywords are different things.

For the next revision, start with the combined exercise requirements and write a fresh playbook. Test an unchanged run as well as a change or failure. The main things to revisit are result structure, booleans and choosing when versus a handler.

## 16. References

1. AU294 AAP 2.5, Chapter 4: Developing Automation Content: Task Control, especially sections 4.1, 4.3 and 4.5. The examples here are based on our home lab practice.
2. [Loops and registered loop results](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html).
3. [Conditionals](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html).
4. [Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html).
5. [Error handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html).
6. [Blocks, rescue and always](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html).

[Back to index](README.md)
