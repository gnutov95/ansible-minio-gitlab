# Ansible MinIO for GitLab

Ansible-роль для автоматизированного развёртывания локального S3-совместимого хранилища **MinIO** на Ubuntu и подключения его к GitLab в качестве объектного хранилища для артефактов, LFS, загрузок и пакетов.

## Возможности

- Установка MinIO из официального бинарника (без Docker).
- Настройка systemd-сервиса с автозапуском и автоперезапуском.
- Создание выделенного системного пользователя `minio`.
- Автоматическое создание бакетов для GitLab:
  - `gitlab-artifacts`
  - `gitlab-lfs`
  - `gitlab-uploads`
  - `gitlab-packages`
  - `gitlab-mr-diffs`
- Поддержка обновления MinIO до новой версии с резервной копией старого бинарника.
- Хранение пароля в Ansible Vault.

## Требования

- Ubuntu 20.04 / 22.04 / 24.04 (или другой дистрибутив с systemd).
- Ansible 2.14+ на машине управления.
- SSH-доступ к целевому серверу с правами `sudo`.
- Свободный порт `9000` (S3 API) и `9001` (веб-консоль).
- Python 3 на целевом хосте (Ansible использует его для модулей).

## Структура проекта

```text
ansible-minio-gitlab/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
│   └── minio_servers/
│       ├── vars.yml
│       └── vault.yml
└── roles/
    └── minio/
        ├── defaults/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   ├── minio.service.j2
        │   └── minio.env.j2
        └── handlers/
            └── main.yml
```

## Быстрый старт

### 1. Клонирование и настройка

```bash
git clone <url-вашего-репозитория> ansible-minio-gitlab
cd ansible-minio-gitlab
```

Отредактируйте `inventory.ini`, указав IP и пользователя целевого сервера:

```ini
[minio_servers]
minio-node1 ansible_host=192.168.1.100 ansible_user=ubuntu
```

### 2. Шифрование пароля

Создайте `group_vars/minio_servers/vault.yml`:

```yaml
vault_minio_root_password: "ВашОченьНадёжныйПароль"
```

И зашифруйте его:

```bash
ansible-vault encrypt group_vars/minio_servers/vault.yml
```

### 3. Запуск установки

```bash
ansible-playbook site.yml --ask-vault-pass
```

После успешного выполнения MinIO будет доступен по адресам:

- S3 API: `http://<IP>:9000`
- Веб-консоль: `http://<IP>:9001`

Логин и пароль — значения `minio_root_user` (по умолчанию `admin`) и `vault_minio_root_password`.

## Обновление MinIO

Обычный запуск playbook **не трогает** установленный бинарник. Чтобы обновить MinIO до актуальной версии:

```bash
ansible-playbook site.yml --ask-vault-pass -e "minio_upgrade=true"
```

Что происходит при обновлении:

1. Текущий бинарник копируется в `/usr/local/bin/minio.bak`.
2. Скачивается свежая версия с `dl.min.io`.
3. systemd-сервис перезапускается.
4. Бакеты и данные сохраняются (они лежат в `/data/minio`).

### Откат

Если после обновления что-то пошло не так, на сервере MinIO:

```bash
sudo systemctl stop minio
sudo mv /usr/local/bin/minio.bak /usr/local/bin/minio
sudo systemctl start minio
```

## Подключение к GitLab

После развёртывания MinIO добавьте в `/etc/gitlab/gitlab.rb` на сервере GitLab:

```ruby
gitlab_rails['object_store']['enabled'] = true
gitlab_rails['object_store']['proxy_download'] = false

gitlab_rails['object_store']['connection'] = {
  'provider' => 'AWS',
  'endpoint' => 'http://<IP_МИНИО>:9000',
  'aws_access_key_id' => 'admin',
  'aws_secret_access_key' => '<пароль из vault>',
  'path_style' => true,
  'region' => 'us-east-1'
}

gitlab_rails['object_store']['objects']['artifacts']['bucket'] = 'gitlab-artifacts'
gitlab_rails['object_store']['objects']['lfs']['bucket'] = 'gitlab-lfs'
gitlab_rails['object_store']['objects']['uploads']['bucket'] = 'gitlab-uploads'
gitlab_rails['object_store']['objects']['packages']['bucket'] = 'gitlab-packages'
gitlab_rails['object_store']['objects']['external_diffs']['bucket'] = 'gitlab-mr-diffs'
```

Примените изменения:

```bash
sudo gitlab-ctl reconfigure
```

> **Важно:** параметр `path_style => true` обязателен для MinIO. Без него GitLab не сможет обращаться к бакетам.

## Переменные

Все переменные задаются в `group_vars/minio_servers/vars.yml`.

| Переменная | Значение по умолчанию | Описание |
|:---|:---|:---|
| `minio_user` | `minio` | Системный пользователь для сервиса |
| `minio_bin_path` | `/usr/local/bin/minio` | Путь к бинарнику MinIO |
| `minio_mc_path` | `/usr/local/bin/mc` | Путь к клиенту `mc` |
| `minio_data_dir` | `/data/minio` | Каталог данных |
| `minio_work_dir` | `/usr/local/minio` | Рабочий каталог |
| `minio_root_user` | `admin` | Логин администратора MinIO |
| `minio_root_password` | из vault | Пароль администратора |
| `minio_buckets` | список бакетов GitLab | Бакеты для создания |
| `minio_upgrade` | `false` | Принудительное обновление бинарника |
| `minio_bin_backup_path` | `/usr/local/bin/minio.bak` | Путь резервной копии |

## Проверка после установки

На сервере MinIO:

```bash
systemctl status minio
/usr/local/bin/minio --version
/usr/local/bin/mc alias set local http://127.0.0.1:9000 admin <пароль>
/usr/local/bin/mc ls local
```

В выводе `mc ls local` должны быть все бакеты из `minio_buckets`.

Из Ansible:

```bash
ansible minio_servers -m command -a "/usr/local/bin/minio --version" --ask-vault-pass
```

## Безопасность

- Пароль администратора хранится только в зашифрованном `vault.yml`.
- Файл `/etc/default/minio` имеет права `0600` и принадлежит пользователю `minio`.
- MinIO работает от непривилегированного системного пользователя.
- Для продакшена рекомендуется настроить TLS (см. ниже).

### TLS (кратко)

Положите сертификаты в `/etc/minio/certs/` (например, `public.crt` и `private.key`), добавьте в `/etc/default/minio`:

```bash
MINIO_OPTS="--console-address :9001 --certs-dir /etc/minio/certs"
```

И в GitLab используйте `https://<IP_МИНИО>:9000`.

## Устранение неполадок

**Сервис не запускается**

```bash
journalctl -u minio -n 100 --no-pager
```

**GitLab не видит бакеты**

- Убедитесь, что `path_style => true` в `gitlab.rb`.
- Проверьте, что бакеты созданы: `mc ls local`.
- Проверьте доступность `http://<IP>:9000` с сервера GitLab.

**Ошибка `Permission denied` при запуске**

Убедитесь, что владелец `/data/minio` и `/usr/local/minio` — пользователь `minio`:

```bash
sudo chown -R minio:minio /data/minio /usr/local/minio
```

## Лицензия

MIT (или укажите свою).