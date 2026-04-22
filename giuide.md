---
layout: page
title: Ansible
permalink: /guide/
description: Ключевые концепции Ansible 
---

## Ansible — ключевые концепции

Шпаргалка по Ansible: архитектура, структура проекта, inventory, playbooks, roles, переменные, Vault и полезные команды.

## Содержание

- [Ansible — ключевые концепции](#ansible--ключевые-концепции)
- [Содержание](#содержание)
- [Архитектура Ansible](#архитектура-ansible)
- [Структура проекта](#структура-проекта)
- [Inventory (INI и YAML)](#inventory-ini-и-yaml)
  - [INI](#ini)
  - [YAML (предпочтительно)](#yaml-предпочтительно)
- [Структура Playbook](#структура-playbook)
- [Ключевые модули (FQCN)](#ключевые-модули-fqcn)
- [Roles](#roles)
- [Ускорение выполнения плейбука](#ускорение-выполнения-плейбука)
- [Приоритет переменных (10 уровней)](#приоритет-переменных-10-уровней)
- [Loops и Conditionals](#loops-и-conditionals)
  - [loop](#loop)
  - [when](#when)
- [Ansible Vault](#ansible-vault)
- [Полезные CLI-команды](#полезные-cli-команды)
- [Tags (включая never)](#tags-включая-never)
- [Jinja2](#jinja2)
- [ansible.cfg (ключевые параметры)](#ansiblecfg-ключевые-параметры)

---

## Архитектура Ansible

Минимальные “кирпичики”:

- **Control Node**: машина, с которой запускается `ansible` / `ansible-playbook`.
- **Managed Nodes**: целевые хосты (Linux/Unix), куда Ansible подключается по SSH и выполняет модули.
- **Inventory**: список хостов и групп, переменные окружения/хостов.
- **Modules**: “операции” (например, `ansible.builtin.package`, `ansible.builtin.template`), которые выполняются на managed node.

Коротко: **Inventory отвечает “где”**, **Playbook/Role — “что”**, **Modules — “как”**.

---

## Структура проекта

Пример “здорового” layout:

```text
ansible-project/
├─ ansible.cfg
├─ inventories/
│  ├─ prod/
│  │  ├─ hosts.yml
│  │  └─ group_vars/
│  │     └─ all.yml
│  └─ stage/
│     ├─ hosts.yml
│     └─ group_vars/
│        └─ all.yml
├─ playbooks/
│  ├─ site.yml
│  └─ web.yml
├─ roles/
│  ├─ common/
│  └─ nginx/
├─ group_vars/
│  └─ all.yml
└─ host_vars/
   └─ web01.yml
```

---

## Inventory (INI и YAML)

### INI

```ini
[web]
web01 ansible_host=10.0.10.11 ansible_user=deploy
web02 ansible_host=10.0.10.12 ansible_user=deploy

[db]
db01 ansible_host=10.0.20.11 ansible_user=postgres

[prod:children]
web
db

[web:vars]
http_port=80
```

### YAML (предпочтительно)

```yaml
all:
  children:
    web:
      hosts:
        web01:
          ansible_host: 10.0.10.11
          ansible_user: deploy
        web02:
          ansible_host: 10.0.10.12
          ansible_user: deploy
    db:
      hosts:
        db01:
          ansible_host: 10.0.20.11
          ansible_user: postgres
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

---

## Структура Playbook

Playbook обычно состоит из одного или нескольких **plays**. У каждого play есть общие “поля” (к каким хостам применять и с какими привилегиями) и секции задач.

Типовая структура (скелет):

```yaml
- name: <что настраиваем>
  hosts: <группа или хосты из inventory>
  become: true|false
  gather_facts: true|false
  serial: <опционально: проценты или число>

  vars:        # переменные play
  vars_files:  # подключаемые файлы с vars (часто + vault)

  pre_tasks:   # подготовка перед ролями/основными задачами
  roles:       # подключение ролей (переиспользуемая логика)
  tasks:       # основные шаги
  post_tasks:  # проверки/очистка/итоговые действия

  handlers:    # действия по notify (обычно перезапуск сервисов)
```

Короткие правила:

- **`pre_tasks`**: проверки и подготовка (пакеты, репозитории, пререквизиты).
- **`roles`**: “переиспользуемые блоки” — то, что хочется выносить и применять в разных playbooks.
- **`tasks`**: основная логика, лучше держать короткими и идемпотентными.
- **`handlers`**: запускаются по `notify`, обычно один раз в конце play (например, `restart nginx`).
- **`serial`**: безопасные изменения “волнами” (rolling update), чтобы не положить весь пул разом.

---

## Ключевые модули (FQCN)

Примеры часто используемых модулей (FQCN обязателен в современном Ansible):

| FQCN | Назначение |
| --- | --- |
| `ansible.builtin.package` | Установка пакетов (универсально) |
| `ansible.builtin.apt` / `ansible.builtin.dnf` | Пакеты на конкретных дистрибутивах |
| `ansible.builtin.template` | Рендер `.j2` шаблонов |
| `ansible.builtin.copy` | Копирование файлов |
| `ansible.builtin.file` | Права/директории/ссылки |
| `ansible.builtin.lineinfile` | Правка строк в файлах |
| `ansible.builtin.service` / `ansible.builtin.systemd` | Управление сервисами |
| `ansible.builtin.command` / `ansible.builtin.shell` | Команды (shell только когда нужно) |
| `ansible.builtin.uri` | HTTP-запросы/проверки |
| `ansible.builtin.user` | Пользователи |
| `ansible.builtin.set_fact` | Вычисляемые переменные |

---

## Roles

Инициализаця структуры роли:

```bash
ansible-galaxy role init roles/nginx
```

Ключевые файлы:

- `roles/<name>/tasks/main.yml`: основная логика
- `roles/<name>/handlers/main.yml`: handler’ы
- `roles/<name>/defaults/main.yml`: **дефолты** (низкий приоритет, удобно переопределять)
- `roles/<name>/vars/main.yml`: **vars** (высокий приоритет, переопределять нежелательно)

---

## Ускорение выполнения плейбука

- Увеличить forks (по умолчанию 5) в ansible.cfg

- Включить pipelining = true — уменьшает количество SSH-соединений

- Использовать strategy: free — хосты не ждут друг друга

- Включить fact_caching — кэшировать факты между запусками

- Отключить сбор фактов где не нужно: gather_facts: false

---

## Приоритет переменных (10 уровней)

Упрощённая таблица (снизу вверх — приоритет растёт):

1. `role defaults` (`roles/*/defaults/main.yml`)
2. `group_vars` (inventory)
3. `host_vars` (inventory)
4. переменные инвентаря (inline в inventory)
5. `group_vars` (playbook)
6. `host_vars` (playbook)
7. `vars:` в play
8. `vars_files:` в play
9. `vars:` в task / block
10. `extra vars` (`-e`) — **всегда самый высокий приоритет**

---

## Loops и Conditionals

### loop

```yaml
- name: Install packages
  ansible.builtin.package:
    name: "${item}"
    state: present
  loop:
    - git
    - curl
    - htop
```

### when

```yaml
- name: Install nginx on Debian
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"
```

---

## Ansible Vault

Основные команды:

```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault decrypt group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
```

Inline-шифрование одной переменной:

```bash
ansible-vault encrypt_string 'SuperSecret123!' --name 'db_password'
```

---

## Полезные CLI-команды

```bash
ansible-playbook playbooks/site.yml --check --diff
ansible-playbook playbooks/site.yml --limit web01
ansible-playbook playbooks/site.yml --tags "nginx,config"
ansible-playbook playbooks/site.yml --skip-tags "verify"

ansible -i inventories/prod/hosts.yml all -m ansible.builtin.ping
ansible-inventory -i inventories/prod/hosts.yml --graph
```

---

## Tags (включая never)

Теги на задаче:

```yaml
- name: Render nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags: [nginx, config]
```

`never` — защита от случайного запуска:

```yaml
- name: Drop database (DANGEROUS)
  ansible.builtin.command: psql -c "DROP DATABASE app;"
  tags: [never, dangerous]
```

Запуск:

```bash
ansible-playbook playbooks/site.yml --tags "never,dangerous"
```

---

## Jinja2

Полезные фильтры (как примеры):

- `default`, `lower`, `upper`, `trim`
- `to_json`, `to_nice_yaml`
- `regex_search`, `regex_replace`

Пример шаблона `.j2`:

```jinja2
# Managed by Ansible
server_name ${inventory_hostname};
listen ${nginx_listen_port | default(80)};
```

---

## ansible.cfg (ключевые параметры)

```ini
[defaults]
inventory = inventories/prod/hosts.yml
forks = 20
roles_path = roles
host_key_checking = False
stdout_callback = yaml
interpreter_python = auto

[connection]
pipelining = True
```
