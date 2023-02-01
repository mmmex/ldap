## LDAP

Задачи:

- [X] Установить FreeIPA;
- [X] [Написать Ansible playbook для конфигурации клиента](#playbook-для-конфигурации-клиента);
- [X] [* Настроить аутентификацию по SSH-ключам](#аутентификация-пользователей-по-ssh-ключу);
- [X] [** Firewall должен быть включен на сервере и на клиенте](#служба-firewalld-на-сервере-и-клиенте).
- [X] Формат сдачи ДЗ - [vagrant](Vagrantfile) + [ansible](ansible/provision.yml)

### Запуск проекта

1. Клонируем репозиторий: `https://github.com/mmmex/ldap.git`
2. Переходим в каталог: `cd ldap`
3. Запускаем проект: `vagrant up`

* Будут подняты и автоматически настроены 2 ВМ:

Хост | IP-адрес (тип интерфейса) | Роль
---|---|---
server.otus.test | 192.168.56.10 (hostonly:vboxnet0); 192.168.57.10 (intnet:clients1-net)| FreeIPA server
client1.otus.test | 192.168.57.11 (intnet:clients1-net) | FreeIPA client

* Зарегистрированные пользователи:

Имя пользователя | Пароль | Роль
---|---|---
admin | Otus2022! | Администратор
i.ivanov | Otus2022! | Пользователь без sudo

* Чтобы выполнить вход в административный UI интерфейс FreeIPA, на хостовой машине (в моем случае Linux Mint) необходимо добавить в файл `/etc/hosts` ip-адрес и имя сервера FreeIPA:

```bash
echo "192.168.56.10 server.otus.test" >> /etc/hosts
```

![image](https://raw.githubusercontent.com/mmmex/ldap/master/screenshots/screenshot1.png)

![image](https://raw.githubusercontent.com/mmmex/ldap/master/screenshots/screenshot2.png)

![image](https://raw.githubusercontent.com/mmmex/ldap/master/screenshots/screenshot3.png)

* Для удаления записи из хостовой машины выполнить:

```bash
sed -i 's/^192.168.56.10 server.otus.test$//' /etc/hosts
```

### Playbook для конфигурации клиента

В скрипт [provision.yml](/ansible/provision.yml) включен фрагмент конфигурации клиента:

```yaml
- name: Configure IPA client
  hosts: client1.otus.test
  become: true
  pre_tasks:
  - name: Enabled firewalld
    ansible.builtin.systemd:
      name: firewalld
      enabled: yes
      state: started
  - name: Add firewalld rules
    ansible.builtin.command:
      firewall-cmd
        --permanent
        --zone=public
        --add-service=freeipa-ldap
        --add-service=freeipa-ldaps
        --add-service=dns
        --add-service=ntp
        --add-service=nfs
    notify: restart firewalld
  roles:
  - role: ipaclient
    state: present
  vars:
    ipaclient_domain: otus.test
    ipaclient_realm: OTUS.TEST
    ipaclient_mkhomedir: true
    ipaclient_force_join: true
    ipasssd_enable_dns_updates: true
    ipaadmin_principal: admin
    ipaadmin_password: Otus2022!
    ipaservers: server.otus.test
    ipaclient_configure_dns_resolver: yes
    ipaclient_dns_servers: 192.168.57.10
    ipaclient_cleanup_dns_resolver: true
  handlers:
  - name: restart firewalld
    ansible.builtin.systemd:
      name: firewalld
      state: restarted
```

_За основу взяты [официальные скрипты ansible](https://github.com/freeipa/ansible-freeipa)_

### Аутентификация пользователей по ssh-ключу

Для демонстрации я сгенерировал ssh ключ пользователя root на `server.otus.test`, далее передаю публичный ключ через register `ssh_key.ssh_public_key` в задачу, которая добавит пользователя `i.ivanov`, фрагмент настройки пользователя:

```yaml
  post_tasks:
  - name: Generate ssh key for root
    user:
      name: root
      generate_ssh_key: yes
      ssh_key_type: ed25519
      ssh_key_file: .ssh/id_ed25519
      state: present
    register: ssh_key
  - name: Create user
    ipauser:
      ipaadmin_password: Otus2022!
      name: i.ivanov
      first: Ivan
      last: Ivanov
      uid: 100001
      gid: 1000
      sshpubkey:
        - "{{ ssh_key.ssh_public_key }}"
      phone: "+7 999 999 99 99"
      email: i.ivanov@otus.test
      password: "Otus2022!"
      update_password: on_create
```

* Для проверки:
  1. Выполняем вход на любую из ВМ, например `server.otus.test`: `vagrant ssh server.otus.test`
  2. Поднимаем привилегии до root: `sudo -i`
  3. Выполняем вход по ssh на `client1.otus.test`: `ssh i.ivanov@client1.otus.test`

```bash
test@test-virtual-machine:~/Otus/ldap$ vagrant ssh server.otus.test
Last login: Thu Feb  2 00:52:30 2023 from 10.0.2.2
[vagrant@server ~]$ sudo -i
[root@server ~]# ssh i.ivanov@client1.otus.test
Creating home directory for i.ivanov.
-sh-4.2$ id
uid=100001(i.ivanov) gid=1000(vagrant) группы=1000(vagrant) контекст=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
-sh-4.2$ ls -al /home/
итого 0
drwxr-xr-x.  4 root     root     37 фев  2 00:53 .
dr-xr-xr-x. 18 root     root    255 фев  2 00:39 ..
drwx------.  2 i.ivanov vagrant  83 фев  2 01:21 i.ivanov
drwx------.  4 vagrant  vagrant  90 фев  2 00:40 vagrant
```

### Служба firewalld на сервере и клиенте

Служба firewalld настроена на всех ВМ. 

* Фрагмент скрипта настройки firewalld на клиенте:

```yaml
  - name: Enabled firewalld
    ansible.builtin.systemd:
      name: firewalld
      enabled: yes
      state: started
  - name: Add firewalld rules
    ansible.builtin.command:
      firewall-cmd
        --permanent
        --zone=public
        --add-service=freeipa-ldap
        --add-service=freeipa-ldaps
        --add-service=dns
        --add-service=ntp
        --add-service=nfs
    notify: restart firewalld
```

* Для сервера включена опция `ipaserver_setup_firewalld`, а также указана зона public `ipaserver_firewalld_zone`, данная конструкция автоматически настраивает firewalld при установке сервера FreeIPA:

```yaml
- name: Prepare server FreeIPA
  hosts: server.otus.test
  become: true
  pre_tasks:
  - name: SELinux set to permissive
    ansible.builtin.selinux:
      policy: targeted
      state: permissive
  roles:
  - role: ipaserver
    state: present
  vars:
    ipadm_password: Otus2022!
    ipaadmin_password: Otus2022!
    ipaserver_ip_addresses:
      - 192.168.56.10
      - 192.168.57.10
      # - 192.168.58.10
    ipaserver_domain: otus.test
    ipaserver_realm: OTUS.TEST
    ipaserver_hostname: server.otus.test
    ipaserver_install_packages: true
    ipaserver_setup_dns: true
    ipaserver_setup_firewalld: true
    ipaserver_firewalld_zone: public
    ipaserver_no_forwarders: true
```

Проверка:
1. Выполним вход на любую ВМ, например `server.otus.test`: `vagrant ssh server.otus.test`
2. Поднимаем привилегии до root: `sudo -i`
3. Проверяем статус работы службы firewalld: `systemctl status firewalld` или `firewall-cmd --state`

```bash
test@test-virtual-machine:~/Otus/ldap$ vagrant ssh server.otus.test
Last login: Thu Feb  2 01:21:26 2023 from 10.0.2.2
[vagrant@server ~]$ sudo -i
[root@server ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Чт 2023-02-02 00:25:00 MSK; 1h 9min ago
     Docs: man:firewalld(1)
 Main PID: 17036 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─17036 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

фев 02 00:25:00 server.otus.test systemd[1]: Starting firewalld - dynamic firewall daemon...
фев 02 00:25:00 server.otus.test systemd[1]: Started firewalld - dynamic firewall daemon.
фев 02 00:25:01 server.otus.test firewalld[17036]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disabling it now.
[root@server ~]# firewall-cmd --state
running
[root@server ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1 eth2
  sources: 
  services: dhcpv6-client dns freeipa-ldap freeipa-ldaps ntp ssh
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```