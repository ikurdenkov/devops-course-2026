# devops-course-2026

Учебный репозиторий дисциплины «Инструменты DevOps».

## Практическая работа № 11. Ansible: роли, идемпотентность, автоматический деплой

Каталог `ansible/` содержит декларативное описание конфигурации управляемого узла
`devops-vm` (Ubuntu Server 24.04):

* `site.yml` — плейбук с ролями `base`, `ssh`, `nginx`, `app`;
* `verify.yml` — read-only аудит развёрнутой конфигурации;
* `inventory/bootstrap.ini` — первичный запуск от `student` по паролю;
* `inventory/prod.ini` — последующие запуски от `devops` по ключу;
* `group_vars/all.yml` — образ GHCR и полный SHA развёрнутой версии.

Документация сервера: [`docs/server-setup.md`](docs/server-setup.md).
