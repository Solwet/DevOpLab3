1. Установка Ansible на управляющую машину (Linux/WSL)
Шаг 1.1: Обновление пакетов системы
sudo apt update
sudo apt upgrade -y
Шаг 1.2: Установка Python и pip
Ansible требует Python 3.9+:

sudo apt install -y python3 python3-pip python3-venv
Проверка версии Python:

python3 --version
Шаг 1.3: Установка Ansible
sudo apt install -y ansible
Или через pip (рекомендуется новая версия):

pip3 install --user ansible
Добавьте pip в PATH (если нужно):

export PATH=$PATH:~/.local/bin
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc
Шаг 1.4: Проверка установки Ansible
ansible --version
Ожидаемый вывод:

ansible [core 2.14.x]
  config file = None
  configured module search path = ['/home/user/.ansible/plugins/modules']
  ...
2. Подготовка SSH ключей для управляемых машин
SSH ключи используются для аутентификации Ansible на управляемых хостах без ввода пароля.

Шаг 2.1: Генерация SSH ключевой пары на управляющей машине
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key -N ""
Параметры:

-t rsa - тип ключа (RSA 4096 бит)
-b 4096 - размер ключа (безопасный размер)
-f ~/.ssh/ansible_key - путь сохранения приватного ключа
-N "" - пустой пароль для ключа (для автоматизации)
Проверка ключей:

ls -la ~/.ssh/ansible_key*
Шаг 2.2: Установка прав доступа на приватный ключ
chmod 600 ~/.ssh/ansible_key
chmod 644 ~/.ssh/ansible_key.pub
3. Запуск управляемого контейнера в Docker
Шаг 3.1: Создание Dockerfile для управляемого хоста
Используйте Dockerfile из раздела "Готовые файлы" ниже.

Шаг 3.2: Создание docker-compose.yml
Используйте docker-compose.yml из раздела "Готовые файлы" ниже.

Шаг 3.3: Сборка и запуск контейнера
# Перейдите в директорию с docker-compose.yml
cd /path/to/project

# Сборка образа
docker-compose build

# Запуск контейнера в фоновом режиме
docker-compose up -d
Шаг 3.4: Проверка запущенного контейнера
docker-compose ps
Ожидаемый вывод:

NAME                    COMMAND               STATUS      PORTS
ansible-managed-host    "/usr/sbin/sshd -D"   Up 1 min    0.0.0.0:2222->22/tcp
Шаг 3.5: Копирование публичного SSH ключа в контейнер
# Создаёте директорию .ssh в контейнере и копируете публичный ключ
docker exec ansible-managed-host mkdir -p /home/ansible/.ssh

docker cp ~/.ssh/ansible_key.pub ansible-managed-host:/home/ansible/.ssh/authorized_keys

# Установка правильных прав доступа
docker exec ansible-managed-host chown -R ansible:ansible /home/ansible/.ssh
docker exec ansible-managed-host chmod 700 /home/ansible/.ssh
docker exec ansible-managed-host chmod 600 /home/ansible/.ssh/authorized_keys
4. Проверка SSH подключения к контейнеру
Шаг 4.1: Проверка SSH подключения
ssh -i ~/.ssh/ansible_key -p 2222 ansible@localhost
Ожидаемый результат: вы должны попасть в bash контейнера без ввода пароля.

Выход из контейнера:

exit
5. Создание инвентарного файла Ansible (inventory)
Инвентарный файл описывает, какие машины управляет Ansible.

Шаг 5.1: Создание файла inventory.ini
Создайте файл inventory.ini в рабочей директории:

[managed_hosts]
managed1 ansible_host=localhost ansible_port=2222 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/ansible_key ansible_python_interpreter=/usr/bin/python3

[all:vars]
ansible_ssh_common_args=-o StrictHostKeyChecking=no
Объяснение параметров:

[managed_hosts] - группа хостов (можно иметь несколько групп)
managed1 - имя хоста в инвентаре (локальное имя, не обязательно реальное)
ansible_host=localhost - IP адрес или FQDN реального хоста
ansible_port=2222 - порт SSH (из docker-compose)
ansible_user=ansible - пользователь для подключения
ansible_ssh_private_key_file - путь к приватному SSH ключу
ansible_python_interpreter - путь к интерпретатору Python на управляемом хосте
ansible_ssh_common_args - отключает проверку ключа хоста (для первого подключения)
Шаг 5.2: Проверка инвентаря
ansible-inventory -i inventory.ini --list
Ожидаемый вывод (JSON формат):

{
    "_meta": {
        "hostvars": {
            "managed1": {
                "ansible_host": "localhost",
                "ansible_port": "2222",
                ...
            }
        }
    },
    "all": {...},
    "managed_hosts": {...},
    "ungrouped": {}
}
6. Проверка подключения Ansible к управляемому хосту
Шаг 6.1: Тест ping
ansible -i inventory.ini managed_hosts -m ping
Ожидаемый вывод:

managed1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
Шаг 6.2: Сбор информации о системе (facts)
ansible -i inventory.ini managed1 -m setup
Выведет всю информацию о системе управляемого хоста.

Шаг 6.3: Выполнение простой команды
ansible -i inventory.ini managed1 -m command -a "uname -a"
Ожидаемый вывод:

managed1 | CHANGED | rc=0 >>
Linux f3a4c8b0c4a2 5.15.0-92-generic #102-Ubuntu SMP Thu Jan 9 10:54:01 UTC 2025 x86_64 GNU/Linux
7. Создание и запуск Ansible Playbook
Шаг 7.1: Структура проекта
project/
├── Dockerfile
├── docker-compose.yml
├── inventory.ini
├── playbook.yml
└── README.md
Шаг 7.2: Создание playbook.yml
Используйте готовый playbook из раздела "Готовые файлы" ниже.

Шаг 7.3: Запуск playbook
# Запуск playbook
ansible-playbook -i inventory.ini playbook.yml
Шаг 7.4: Вывод playbook
PLAY [managed_hosts] ************************************************************

TASK [Gathering Facts] **********************************************************
ok: [managed1]

TASK [Update package list] ******************************************************
changed: [managed1]

TASK [Install required packages] ************************************************
changed: [managed1]

TASK [Create test directory] ****************************************************
changed: [managed1]

TASK [Create test file with content] ********************************************
changed: [managed1]

TASK [Display file content] *****************************************************
ok: [managed1] => {
    "msg": "File content from managed host: Hello from Ansible!\nThis is a test file created by Ansible playbook."
}

TASK [Get system information] ***************************************************
ok: [managed1] => {
    "msg": "System: Linux, Hostname: f3a4c8b0c4a2, Uptime: 12 min"
}

PLAY RECAP ************************************************************
managed1 : ok=7 changed=4 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
