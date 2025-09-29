---
- name: Image / host software + OS audit
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    audit_local_dir: "./audit_results"   # where fetched JSON files will be placed on control node
  tasks:
    - name: Ensure local fetch directory exists (control node)
      local_action:
        module: file
        path: "{{ audit_local_dir }}"
        state: directory
        mode: '0755'

    - name: Collect package metadata with package_facts (works for apt/yum/dnf)
      ansible.builtin.package_facts:
        manager: auto
      register: pkg_facts
      failed_when: false
      changed_when: false

    - name: Run misc detection commands (safe: non-fatal)
      vars:
        cmds:
          uname: "uname -a"
          os_release: "cat /etc/os-release || true"
          lsb_release: "lsb_release -a || true"
          rpm_list: "rpm -qa || true"
          dpkg_list: "dpkg-query -W -f='${Package} ${Version}\\n' || true"
          pip3_list: "python3 -m pip list --format=json || true"
          pip_list: "python -m pip list --format=json || true"
          npm_global: "npm -g ls --depth=0 --json || true"
          gem_list: "gem list --local || true"
          docker_version: "docker --version || true"
          podman_version: "podman --version || true"
          containerd_version: "containerd --version || true"
          systemctl_services: "systemctl list-units --type=service --all --no-pager --no-legend || true"
          listening_ports_ss: "ss -tunlp || true"
          listening_ports_netstat: "netstat -tunlp || true"
          lsmod: "lsmod || true"
          mounts: "mount | column -t || true"
          df: "df -h || true"
          users: "getent passwd || true"
          root_crontab: "crontab -l -u root || true"
          cron_spool: "ls -la /var/spool/cron* || true"
          snap_list: "snap list || true"
          flatpak_list: "flatpak list || true"
          virt_detect: "systemd-detect-virt || true"
      block:
        - name: Run detection commands and register outputs
          ansible.builtin.shell: "{{ item.value }}"
          args:
            warn: false
          loop: "{{ cmds | dict2items }}"
          loop_control:
            label: "{{ item.key }}"
          register: cmd_results
          failed_when: false
          changed_when: false

    - name: Map command results to a dictionary
      ansible.builtin.set_fact:
        cmd_output: >-
          {{
            dict(
              cmd_results.results | map('extract', attribute='stdout', default='') 
              | zip(cmd_results.results | map(attribute='item.key'))
            )
          }}
      changed_when: false

    - name: Collect service facts (systemd) (non-fatal)
      ansible.builtin.shell: "systemctl list-unit-files --type=service --no-pager --no-legend || true"
      register: unit_files
      failed_when: false
      changed_when: false

    - name: Create the audit dictionary
      ansible.builtin.set_fact:
        image_audit:
          hostname: "{{ ansible_facts['nodename'] | default(inventory_hostname) }}"
          inventory_hostname: "{{ inventory_hostname }}"
          ansible_facts_summary:
            os_family: "{{ ansible_facts['os_family'] | default('unknown') }}"
            distribution: "{{ ansible_facts['distribution'] | default(ansible_facts['ansible_distribution'] | default('unknown')) }}"
            distribution_version: "{{ ansible_facts['distribution_version'] | default(ansible_facts['ansible_distribution_version'] | default('unknown')) }}"
            kernel: "{{ ansible_facts['kernel'] | default('unknown') }}"
            architecture: "{{ ansible_facts['architecture'] | default(ansible_facts['ansible_architecture'] | default('unknown')) }}"
            python_version: "{{ ansible_facts['python']['version']['string'] if (ansible_facts.python is defined and ansible_facts.python.version is defined) else (ansible_facts['ansible_python_version'] | default('unknown')) }}"
            ip_addresses: "{{ ansible_facts['all_ipv4_addresses'] | default([]) }}"
          package_facts: "{{ pkg_facts.packages | default({}) }}"
          command_outputs: "{{ cmd_output }}"
          unit_files: "{{ unit_files.stdout | default('') }}"
          additional_notes: "Some outputs may be empty if utilities are not installed on the target."
      changed_when: false

    - name: Save audit JSON on remote host (/tmp)
      ansible.builtin.copy:
        dest: "/tmp/image_audit_{{ inventory_hostname }}.json"
        content: "{{ image_audit | to_nice_json }}"
        mode: '0644'
      register: remote_write
      changed_when: false

    - name: Fetch audit file to control machine
      ansible.builtin.fetch:
        src: "/tmp/image_audit_{{ inventory_hostname }}.json"
        dest: "{{ audit_local_dir }}/"
        flat: yes
      register: fetched
      failed_when: false

    - name: Show short summary
      ansible.builtin.debug:
        msg:
          - "Saved remote audit to /tmp/image_audit_{{ inventory_hostname }}.json"
          - "Fetched to {{ fetched.dest if fetched is defined and fetched.dest is not none else (audit_local_dir ~ '/' ~ inventory_hostname ~ '_image_audit_{{ inventory_hostname }}.json') }}"
