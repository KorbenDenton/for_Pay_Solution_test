# Тестовое задание для Pay Solution
Выполнил: Зайнуллин Айзат Маратович
## Ansible Server Configuration

Комплексное решение для приведения чистого сервера Ubuntu 20.04/22.04 к безопасному
базовому состоянию с подключением к мониторингу и управлением секретами через
Ansible Vault.

## 📋 Что делает этот плейбук

### Часть A — Базовая конфигурация сервера (роль `configuration`)
- Создаёт пользователя `ops` с правами `sudo` и доступом по SSH-ключу
- Настраивает SSH: запрещает вход по паролю и под root, разрешает только `ops`
- Включает UFW с политикой deny-by-default и разрешает только порт 22/tcp
- Устанавливает базовые пакеты: `fail2ban`, `auditd`, `curl`, `jq`

### Часть B — Агент мониторинга (роль `node_exporter`)
- Устанавливает `node_exporter` как systemd-сервис
- По умолчанию слушает `127.0.0.1:9100` (только localhost)
- При необходимости порт открывается для IP сервера Prometheus

### Часть C — Управление секретами (роль `example_app`)
- Демонстрирует работу с `ansible-vault` для защиты чувствительных данных
- Шаблонизирует конфиг `/etc/example-app/config.env` с правами `0600`
- Исключает утечку секретов в вывод Ansible через `no_log: true`


## Быстрый старт

### Предварительные требования

- Ansible 2.15+ установлен на управляющей машине
- Целевой сервер: чистая Ubuntu 20.04/22.04
- Доступ к серверу по SSH с правами sudo

### 1. Подготовка инвентаря

Отредактируй `inventory.ini`:

```ini
[server]
your-server-ip ansible_user=your_user
```

### 2. Настройка SSH-ключа для пользователя ops

Положи свой публичный ключ в файл роли `configuration`:

```bash
cp ~/.ssh/id_rsa.pub roles/configuration/files/authorized_keys
```

Либо создай файл вручную и вставь содержимое публичного ключа.

### 3. Настройка секретов Vault

Создай файл с паролем для Vault:

```bash
echo "MyV3ryStr0ngVaultP4ssw0rd!" > vault-password-file
chmod 600 vault-password-file
```

Создай и зашифруй `group_vars/all/vault.yml`:

```bash
# Создай файл с секретами (открытый вид)
cat > group_vars/all/vault.yml << EOF
---
vault_app_token: "s3cr3t-t0k3n-f0r-ex4mpl3-4pp"
vault_db_password: "sup3r-s3cur3-p4ssw0rd"
EOF

# Зашифруй его
ansible-vault encrypt group_vars/all/vault.yml --vault-password-file vault-password-file
```

### 4. Запуск плейбука

#### Полный запуск со всеми ролями

```bash
ansible-playbook -i inventory.ini play_roles.yaml \
  --vault-password-file vault-password-file \
  --ask-become-pass
```

#### Запуск с интерактивным запросом пароля Vaultinventory/inventory.ini

```bash
ansible-playbook -i inventory.ini play_roles.yaml \
  --ask-vault-pass \
  --ask-become-pass
```

#### Запуск отдельных частей (по тегам)

```bash
# Только базовая настройка сервера
ansible-playbook -i inventory.ini play_roles.yaml \
  --tags base \
  --ask-become-pass

# Только установка node_exporter
ansible-playbook -i inventory.ini play_roles.yaml \
  --tags monitoring \
  --ask-become-pass

# Только развертывание конфига с секретами
ansible-playbook -i inventory.ini play_roles.yaml \
  --tags app \
  --vault-password-file vault-password-file \
  --ask-become-pass
```

### 5. Открытие порта для Prometheus (опционально)

По умолчанию `node_exporter` слушает только на `localhost` (`127.0.0.1:9100`).

Если Prometheus находится на отдельном сервере, добавь переменную в инвентарь
или плейбук:

```yaml
monitoring_server_ip: "192.168.1.100"  # IP твоего Prometheus
```

Порт 9100 будет открыт только для этого IP.

## Проверка результатов

### Часть A — Базовая конфигурация

```bash
# Проверка пользователя ops
ssh ops@your-server-ip  # Должен работать вход по ключу
sudo -l -U ops           # Должны быть права NOPASSWD: ALL

# Проверка конфигурации SSH
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|allowusers"
# Ожидаемый вывод:
# permitrootlogin no
# passwordauthentication no
# allowusers ops

# Проверка статуса fail2ban и auditd
sudo systemctl status fail2ban
sudo systemctl status auditd
# Оба должны быть active (running)

# Проверка установки пакетов
dpkg -l | grep -E "curl|jq"
# Оба должны быть в состоянии ii (installed)

# Проверка UFW
sudo ufw status verbose
# Ожидаемый вывод:
# Status: active
# Default: deny (incoming), allow (outgoing)
# 22/tcp                     ALLOW IN    Anywhere
```

### Часть B — node_exporter

```bash
# Проверка статуса сервиса
sudo systemctl status node_exporter
# Должен быть active (running) и enabled

# Проверка, что метрики доступны на localhost
curl -s http://localhost:9100/metrics | head -20
# Должен показать список метрик Prometheus

# Проверка, что порт НЕ доступен снаружи (если не настраивали)
# Выполнить с другого хоста:
curl -s --connect-timeout 5 http://your-server-ip:9100/metrics
# Должен упасть с таймаутом или "Connection refused"
```

### Часть C — Конфиг с секретами

```bash
# Проверка прав доступа
ls -la /etc/example-app/config.env
# Должен показать: -rw------- 1 root root ... /etc/example-app/config.env

# Проверка содержимого
sudo cat /etc/example-app/config.env
# Должен содержать значения секретов (не имена переменных)
```

### Идемпотентность

```bash
# Повторный запуск плейбука не должен показывать изменений
ansible-playbook -i inventory.ini play_roles.yaml \
  --vault-password-file vault-password-file \
  --ask-become-pass

# Все задачи должны иметь статус ok (не changed), кроме первой проверки
```

## Работа с Ansible Vault

```bash
# Просмотр зашифрованного файла
ansible-vault view group_vars/all/vault.yml \
  --vault-password-file vault-password-file

# Редактирование секретов
ansible-vault edit group_vars/all/vault.yml \
  --vault-password-file vault-password-file

# Смена пароля Vault
ansible-vault rekey group_vars/all/vault.yml \
  --vault-password-file vault-password-file

# Шифрование отдельной переменной (для CI/CD)
ansible-vault encrypt_string "secret_value" --name "variable_name"
```

## Безопасность

- Все секреты зашифрованы через `ansible-vault`
- Файл `vault-password-file` исключён из Git через `.gitignore`
- Конфигурационные файлы с секретами имеют права `0600` и владельца `root`
- Вывод секретов в логи Ansible заблокирован через `no_log: true`
- SSH доступ только по ключу, root-login запрещён
- UFW в режиме deny-by-default, открыт только порт 22

## Обновление node_exporter

Для обновления версии измени переменную в `roles/observability/defaults/main.yml`:

```yaml
node_exporter_version: "1.8.3"  # новая версия
```

При следующем запуске плейбук автоматически скачает и установит новую версию.
```