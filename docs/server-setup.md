# Настройка сервера

Документ фиксирует фактическое состояние учебного стенда, на котором развёрнуто
приложение `devops-ci-app`. Все значения приведены по результатам проверок на самих
узлах.

## 1. Общие сведения

| Параметр | Значение |
|---|---|
| Управляемый узел | `devops-vm`, Ubuntu Server 24.04 LTS, amd64 |
| Управляющий узел | `devops-control`, Ubuntu Server 24.04 LTS, amd64 |
| Адрес управляемого узла | `192.168.56.101` |
| Адрес управляющего узла | `192.168.56.102` |
| Сегмент | `192.168.56.0/24`, VirtualBox NAT Network |

## 2. Учётные записи

* `student` — первичная учётная запись, созданная при установке; используется только
  для bootstrap-запуска Ansible по паролю.
* `devops` — учётная запись автоматизации, вход только по ключу `~/.ssh/devops_vm`,
  sudo без пароля (файл `/etc/sudoers.d/devops`).

## 3. Службы

| Служба | Назначение | Состояние |
|---|---|---|
| `ssh` | удалённое управление, порт 22 | enabled, running |
| `docker` | контейнерная среда | enabled, running |
| `nginx` | reverse proxy, порт 80 | enabled, running |

## 4. Сеть и межсетевой экран

* UFW: входящие по умолчанию запрещены, разрешены TCP 22 и 80, исходящие разрешены.
* Порт приложения публикуется только на loopback: `127.0.0.1:5000`.
* Наружу приложение доступно исключительно через Nginx на порту 80.

## 5. Docker

Docker Engine установлен из официального репозитория
`https://download.docker.com/linux/ubuntu` (пакеты `docker-ce`, `docker-ce-cli`,
`containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`).

## 6. Nginx

Конфигурация сайта: `/etc/nginx/sites-available/devops-ci-app`, включена символической
ссылкой в `sites-enabled`, сайт по умолчанию отключён. Проксирование на
`http://127.0.0.1:5000` с передачей заголовков `Host`, `X-Real-IP`, `X-Forwarded-For`,
`X-Forwarded-Proto`.

## 7. Приложение

| Параметр | Значение |
|---|---|
| Образ | `ghcr.io/ikurdenkov/devops-ci-app` |
| Контейнер | `devops-ci-app` |
| Порт контейнера | 5000 |
| Публикация | `127.0.0.1:5000:5000` |
| Health endpoint | `GET /health` → `{"status":"ok"}` |
| Политика перезапуска | `unless-stopped` |

## 8. Проверки

```bash
curl --fail http://192.168.56.101/health
curl --fail http://192.168.56.101/
sudo ss -tlnp | grep -E ':(22|80|5000)\b'
docker ps --filter name=devops-ci-app
sudo ufw status verbose
```

## 9. Диагностика

Порядок разбора отказа: состояние runner → доступность SSH → результат Ansible →
состояние контейнера → ответ Nginx. Для каждого звена используется отдельная команда,
чтобы не подменять причину симптомом.

## 10. Ansible

**Расположение проекта.** Каталог `ansible/` в репозитории `devops-ci-app`
(в практической работе № 11 — `ansible/` в репозитории `devops-course-2026`).

**Управляющий и управляемый узлы.**

* управляющий узел — `devops-control` (`192.168.56.102`): Ansible, коллекции,
  ключ `~/.ssh/devops_vm`, self-hosted runner GitHub Actions;
* управляемый узел — `devops-vm` (`192.168.56.101`): целевая конфигурация.

**Inventory.**

* `ansible/inventory/bootstrap.ini` — узел `devops-vm`, пользователь `student`,
  подключение по паролю (`--ask-pass`), применяется только при первом запуске;
* `ansible/inventory/prod.ini` — тот же узел, пользователь `devops`, ключ
  `~/.ssh/devops_vm`, применяется при всех последующих запусках.

Файл `ansible/ansible.cfg` выбирает `inventory/prod.ini` по умолчанию, поэтому
`ansible-playbook site.yml` без `-i` работает по ключу и не требует интерактивного ввода.

**Учётные записи.**

| Этап | Пользователь | Аутентификация | Команда |
|---|---|---|---|
| bootstrap | `student` | пароль + sudo по паролю | `ansible-playbook -i inventory/bootstrap.ini site.yml --ask-pass --ask-become-pass` |
| prod | `devops` | ключ `~/.ssh/devops_vm` + sudo без пароля | `ansible-playbook site.yml` |

**Роли и ответственность.**

| Роль | Ответственность |
|---|---|
| `base` | проверка платформы, базовые пакеты, репозиторий и служба Docker |
| `ssh` | пользователь `devops`, ключ, sudoers, hardening SSH, UFW |
| `nginx` | пакет Nginx, reverse proxy, включение сайта, проверка конфигурации |
| `app` | загрузка образа по SHA, контейнер, проверка `/health` |

**Адрес образа GHCR.** `ghcr.io/ikurdenkov/devops-ci-app` (владелец указан строчными
буквами).

**Полный SHA.** `0f9ca2125f034a2cfdbdb5779ef50a16161f887c` — коммит, для которого
конвейер работы № 10 опубликовал образ; задан переменной `app_tag` в
`ansible/group_vars/all.yml`. В контрольной работе № 3 значение передаётся параметром
`-e app_tag=$GITHUB_SHA` и файл не редактируется.

**Порт контейнера.** Внутренний 5000, публикация `127.0.0.1:5000:5000`.

**Health endpoint.** `GET http://127.0.0.1:5000/health`, ожидается
`{"status":"ok"}`.

**Команды.**

```bash
# первичная настройка
cd ansible
ansible-playbook -i inventory/bootstrap.ini site.yml --ask-pass --ask-become-pass

# повторный запуск и деплой
ansible-playbook site.yml
ansible-playbook site.yml -e app_tag=<полный-SHA>

# проверка синтаксиса без обращения к узлу
ansible-playbook site.yml --syntax-check
```

**Read-only аудит.**

```bash
ansible-playbook verify.yml
```

Плейбук `verify.yml` не изменяет конфигурацию: проверяет действующие параметры `sshd`,
факт запуска контейнера и ответ `/health`, поэтому его результат можно использовать как
проверку в CI.

**Пароли и закрытые ключи.** В документ и в репозиторий они не включаются. Закрытый ключ
`~/.ssh/devops_vm` хранится только в домашнем каталоге управляющего узла с правами
`0600`.
