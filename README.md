# Домашнее задание № 25 по теме: "LDAP. Централизованная авторизация и аутентификация". К курсу Administrator Linux. Professional

## Задание

- Установить FreeIPA
- Написать Ansible-playbook для конфигурации клиента

- \* Настроить аутентификацию по SSH-ключам
- \*\* Firewall должен быть включен на сервере и на клиенте

### Выполнение

- Дополнительно написаны задачи для автоматического развёртывания FreeIPA server
- SELinux **не** отключен
- Реализован доступ по SSH-ключам. **Открытая часть ключевой пары берется у текущего пользователя (под которым разворачивается стенд) ~/.ssh/id_rsa.pub**

- Пароль admin и otus-user: **otus2023**

nftables.conf на ipa.otus.lan:
```
table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                iif "lo" accept
                ct state invalid counter drop
                ct state established,related accept
                ct state new tcp dport 22 accept comment "SSH accept"
                ct state new tcp dport { 80, 443 } counter accept comment "HTTP(S) access for management IPA"
                ct state new tcp dport { 389, 636 } counter accept comment "LDAP/LDAPS"
                ct state new tcp dport { 88, 464 } counter accept comment "kerberos"
                ct state new udp dport { 88, 464 } counter accept comment "kerberos"
                ct state new udp dport 123 counter accept comment "NTP"
        }
}
```

nftables.conf на клиентах:
```
table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                ct state invalid counter packets 0 bytes 0 drop
                iif "lo" accept
                ct state new tcp dport 22 accept
                ct state established,related accept
                ip protocol icmp accept
                udp dport 33434-33524 counter packets 0 bytes 0 accept comment "for traceroute"
        }
}
```

### Проверка

```bash
vagrant ssh ipa.otus.lan -c 'sestatus'
```
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      32
```

```bash
vagrant ssh client1.otus.lan -c 'sestatus'
```
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      32
```

```bash
ssh otus-user@192.168.56.11
```
```
Last login: Thu Mar 28 22:38:44 2024 from 192.168.56.101
[otus-user@client1 ~]$
```

- Проверка выдачи билета
```bash
kinit
```
```
Password for otus-user@OTUS.LAN:
Password expired.  You must change it now.
Enter new password:
Enter it again:
```

- Проверка выданного билета
```bash
klist
```
```
Ticket cache: KCM:1120800003
Default principal: otus-user@OTUS.LAN

Valid starting       Expires              Service principal
03/28/2024 22:43:22  03/29/2024 22:43:22  krbtgt/OTUS.LAN@OTUS.LAN
```

**Playbook**
```yaml
---
- hosts: all
  become: true
  gather_facts: true

  tasks:
  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos
    when: (ansible_os_family == "RedHat")

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"
    when: (ansible_os_family == "RedHat")

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"
    when: ansible_os_family == "RedHat"

  - name: Install epel-release
    ansible.builtin.yum:
      name: epel-release
      state: present
    when: (ansible_os_family == "RedHat")

  - name: Install soft on RedHat-based OS
    ansible.builtin.yum:
      name:
        - vim
        - tcpdump
        - traceroute
        - net-tools
      state: present
      update_cache: true
    when: ansible_os_family == "RedHat"

  handlers:

  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: restarted

- hosts: ipa.otus.lan
  become: true
  gather_facts: false

  vars:
    rsa_pub: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

  tasks:

  - name: Copy nftables config on ipa.otus.lan
    ansible.builtin.copy:
      src: files/ipa.otus.lan-nftables.conf
      dest: /etc/nftables.conf
    notify:
      - Configure nftables

  - name: Install IPA server
    ansible.builtin.yum:
      name:
        - '@idm:DL1'
        - ipa-server

  - name: Edit /etc/hosts
    ansible.builtin.replace:
      path: /etc/hosts
      regexp: '127\.\d\.\d\.\d (ipa.+)'
      replace: '192.168.56.10 \1'

  - name: Deploy isa server and create user otus-user
    ansible.builtin.shell: "{{ item }}"
    with_items:
      - 'echo -e "no\nyes" | ipa-server-install -n otus.lan -r OTUS.LAN --hostname=ipa.otus.lan --no-ntp --netbios-name=OTUS -p otus2023 -a otus2023'
      - 'echo "otus2023" | kinit admin'
      - 'echo "otus2023" | ipa user-add otus-user --first=Otus --last=User --password --sshpubkey="{{ rsa_pub }}"'

  handlers:

  - name: Configure nftables
    ansible.builtin.lineinfile:
      path: /etc/sysconfig/nftables.conf
      line: 'include "/etc/nftables.conf"'
      state: present
    notify:
      - Restart nftables service

  - name: Restart nftables service
    ansible.builtin.service:
      name: nftables
      enabled: true
      state: restarted


- hosts: clients
  become: true
  gather_facts: false

  tasks:

  - name: Add ipa.otus.lan to /etc/hosts
    ansible.builtin.lineinfile:
      path: /etc/hosts
      line: 192.168.56.10 ipa.otus.lan ipa

  - name: Install freeipa-client
    ansible.builtin.yum:
      name: freeipa-client
      state: present
      update_cache: true

  - name: Add host to ipa-server
    shell: echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w otus2023

  - name: Copy nftables config on client
    ansible.builtin.copy:
      src: files/clients-nftables.conf
      dest: /etc/nftables.conf
    notify:
      - Configure nftables

  handlers:

  - name: Configure nftables
    ansible.builtin.lineinfile:
      path: /etc/sysconfig/nftables.conf
      line: 'include "/etc/nftables.conf"'
      state: present
    notify:
      - Restart nftables service

  - name: Restart nftables service
    ansible.builtin.service:
      name: nftables
      enabled: true
      state: restarted
```

