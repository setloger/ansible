---
layout: page
title: "Презентация: роль"
description: "Кратко о роли в репозитории и процессе reboot через Ansible"
permalink: /presentation-my-role/
---

<!-- markdownlint-disable -->

# Презентация: роль в репозитории `ansible-reboot-nodes`

---

- **Цель репозитория**: безопасно и предсказуемо **перезагружать хосты по inventory**, с понятной сериализацией, контролем готовности после reboot и **уведомлениями**.

---

## Что я сделал

- **Собрал точку входа**: плейбук `ansible-reboot-main/play-reboot-hosts.yml`.
- **Оформил роль ребута**: `ansible-reboot-main/roles/reboot/` с понятной последовательностью шагов.
- **Добавил уведомления по email** (опционально, без остановки ребута).

---

## Архитектура: из чего состоит ребут-прогон

Роль `reboot` выполняет шаги:

- **Preflight**: проверки/подготовка к прогону.
- **Facts before**: фиксация состояния “до”.
- **Notify start**: письмо о старте (тег `notify_start`).
- **Reboot**: перезагрузка через `ansible.builtin.reboot`.
- **Facts after**: фиксация состояния “после”.
- **Notify end**: письмо об окончании.
- **Report to localhost**: локальный отчёт на control node.

---

## Безопасность и управляемость: как мы избегаем “массового” ребута

- **Сериализация** через `serial` в плейбуке: `reboot_serial | default(1)`.
- **Контроль** через `--limit` (точечно) и inventory (группы/хосты).
- **Поведение при проблемах с почтой**: уведомления не блокируют ребут (предупреждение, продолжается прогон).

---

## Уведомления по email: ключевая идея

- В ряде окружений SMTP (порт 25) доступен **не со всех хостов**.
- Поэтому отправку делаем через **delegate-хосты**: `notify_delegate_hosts`.
- Логика: **перебор delegate-хостов до первого успеха**, затем дальше.

---

## Уведомления: основные переменные

Включение/адреса:

- `notify_email_enabled`: включить/выключить отправку.
- `notify_email_from`: адрес отправителя.
- `notify_email_recipients`: список получателей.

Где отправлять (важно):

- `notify_delegate_hosts`: список хостов, с которых пробуем отправить письмо.
- `notify_delegate_user` : логин для delegate.

Формат:

- `notify_email_subtype`: `plain` или `html`.
- `notify_email_template`: шаблон письма (по умолчанию используются шаблоны роли).

SMTP:

- `smtp_host`, `smtp_port`.

---

## Контроль готовности после перезагрузки (важно для стабильности)

Для `ansible.builtin.reboot` в роли предусмотрены:

- `reboot_timeout`: общий таймаут ожидания после reboot.
- `reboot_boot_time_command`: команда определения “времени/возраста загрузки” (может помочь в нестандартных окружениях).
- `reboot_test_command`: команда проверки готовности хоста после reboot (например, `uptime`).

---

## Как запускать (быстрый старт)

Рекомендуемый вариант — из каталога `ansible-reboot-main`:

```bash
cd ansible-reboot-main
ansible-playbook -i inventories/<env>/inv.yml play-reboot-hosts.yml
```

Ограничить список хостов:

```bash
cd ansible-reboot-main
ansible-playbook -i inventories/<env>/inv.yml play-reboot-hosts.yml --limit 'host1,host2'
```

---

## Где живут настройки

- **Глобальные дефолты**: `ansible-reboot-main/group_vars/all.yml`
- **Окружение (env)**: `ansible-reboot-main/inventories/<env>/group_vars/all.yml`
- **Дефолты роли**: `ansible-reboot-main/roles/reboot/defaults/main.yml`

---

## Дальше (куда развивать)

- **Kubernetes‑режим** : drain → reboot → uncordon, проверки kubectl get node, порядок групп (reboot_group_order), отдельные политики для групп (db/redis/infra) (почему бы нет)
- **Тестирование**: Molecule/CI‑проверки (как вариант, интеграция в А-платформа)
- **Наблюдаемость** кроме почты: webhook/messenger, интеграция с мониторингом (как варианта)


