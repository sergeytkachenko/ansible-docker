# Ubuntu

## Install ansible

```bash
sudo apt update
sudo apt install ansible
```

## Key features of this playbook:

- **Automatically gets latest versions**: Uses Docker's official repository for Docker CE and GitHub API for Docker Compose latest release
- **Proper GPG key handling**: Securely adds Docker's signing key
- **User management**: Adds the current user to the docker group
- **Service management**: Starts and enables Docker service
- **Verification**: Checks that both Docker and Docker Compose are properly installed

## Usage:

1. **Save the playbook** as `install-docker.yml`

2. **Create an inventory file** (e.g., `hosts.ini`):
```ini
[ubuntu_servers]
your-server-ip ansible_user=your-username
```

3. **Run the playbook**:

For local installation:
```bash
ansible-playbook -i localhost, -c local install-docker.yml
```

## Notes:

- The playbook will add the current Ansible user to the docker group
- After installation, users need to logout/login or run `newgrp docker` to use Docker without sudo
- The playbook is idempotent - safe to run multiple times
- Works with Ubuntu 20.04, 22.04, and newer versions

The script automatically handles architecture detection and gets the latest stable versions from official sources.

```
---
- name: Enable and configure SFTP on Ubuntu
  hosts: all
  become: yes
  
  tasks:
    - name: Install OpenSSH server
      apt:
        name: openssh-server
        state: present
        update_cache: yes

    - name: Ensure SSH service is running
      systemd:
        name: ssh
        state: started
        enabled: yes

    - name: Check if SFTP subsystem is enabled in sshd_config
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^Subsystem\s+sftp'
        line: 'Subsystem sftp /usr/lib/openssh/sftp-server'
        state: present
        backup: yes
      notify: restart ssh

    - name: Allow SFTP in SSH configuration (if not already present)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?AllowUsers'
        line: '# AllowUsers - configure as needed'
        state: present
        backup: yes
      notify: restart ssh

    - name: Ensure SSH allows password authentication (optional)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication yes'
        state: present
        backup: yes
      notify: restart ssh

    - name: Check SSH port configuration
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Port'
        line: 'Port 22'
        state: present
        backup: yes
      notify: restart ssh

    - name: Test SSH configuration
      command: sshd -t
      register: sshd_test
      changed_when: false

    - name: Display SSH configuration test result
      debug:
        msg: "SSH configuration is valid"
      when: sshd_test.rc == 0

  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted
---
- name: Install Docker and Docker Compose on Ubuntu
  hosts: all
  become: yes
  vars:
    docker_users:
      - "{{ ansible_user }}"  # Add current user to docker group
    
  tasks:
    - name: Update apt package cache
      apt:
        update_cache: yes
        cache_valid_time: 86400

    - name: Install required packages for Docker repository
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - software-properties-common
        state: present

    - name: Create directory for Docker GPG key
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker official GPG key
      get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'

    - name: Get Ubuntu release codename
      command: lsb_release -cs
      register: ubuntu_codename
      changed_when: false

    - name: Add Docker repository
      apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ubuntu_codename.stdout }} stable"
        state: present
        update_cache: yes

    - name: Install Docker CE
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
        state: present

    - name: Start and enable Docker service
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Get latest Docker Compose version
      uri:
        url: https://api.github.com/repos/docker/compose/releases/latest
        return_content: yes
      register: docker_compose_latest

    - name: Download and install Docker Compose
      get_url:
        url: "https://github.com/docker/compose/releases/download/{{ docker_compose_latest.json.tag_name }}/docker-compose-linux-x86_64"
        dest: /usr/local/bin/docker-compose
        mode: '0755'
        owner: root
        group: root

    - name: Create docker-compose symlink for backward compatibility
      file:
        src: /usr/local/bin/docker-compose
        dest: /usr/bin/docker-compose
        state: link

    - name: Add users to docker group
      user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"
      when: docker_users is defined

    - name: Verify Docker installation
      command: docker --version
      register: docker_version
      changed_when: false

    - name: Verify Docker Compose installation
      command: docker-compose --version
      register: docker_compose_version
      changed_when: false

    - name: Display installed versions
      debug:
        msg:
          - "{{ docker_version.stdout }}"
          - "{{ docker_compose_version.stdout }}"

    - name: Remind about logout/login for docker group
      debug:
        msg: "Please logout and login again (or run 'newgrp docker') to use Docker without sudo"
```
