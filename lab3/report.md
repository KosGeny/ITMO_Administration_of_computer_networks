# Лабораторная работа №3: Ansible + Caddy  
---
## **Часть 1. Установка и настройка Ansible**

### 1. Установка менеджера пакетов pip для Python

`curl https://bootstrap.pypa.io/ get-pip.py -o get-pip.py && python3 get-pip.py`

<img width="817" height="186" alt="1" src="https://github.com/user-attachments/assets/1c91f816-41e9-492c-8612-31ec0607b490" />

### 2. Установка системы управления конфигурациями Ansible 

`python3 -m pip install ansible`

<img width="867" height="499" alt="2" src="https://github.com/user-attachments/assets/010d4b96-4990-4b2b-8540-e281e3fa6779" />

### 3. Создание базовой структуры конфигурационных файлов Ansible

Создаем рабочий каталог с конфигурационным файлом ansible.cfg и подкаталог inventory для хранения информации о целевых хостах.

<img width="643" height="44" alt="3_1" src="https://github.com/user-attachments/assets/2ee7e20b-886f-4186-af52-43e618686a89" />
<img width="734" height="58" alt="3_2" src="https://github.com/user-attachments/assets/22667f39-33da-4fb1-af9c-d18be157fb73" />
<img width="756" height="40" alt="3_3" src="https://github.com/user-attachments/assets/5d25ab6b-b1c2-4e2f-8820-45fe66404f38" />

###  4. Проверка работоспособности Ansible и подключения к целевым хостам

`ansible my_servers -m ping -c local`

<img width="867" height="186" alt="4_1" src="https://github.com/user-attachments/assets/db731089-9a53-4520-978b-b200c066e010" />

`ansible my_servers -m setup -c local`

<img width="867" height="470" alt="4_2" src="https://github.com/user-attachments/assets/d4154ceb-de4c-47ba-93b7-05617a76cfa8" />

### 5. По заданию лабораторной работы: 
#### Разрабатываем единый плейбук file_management.yml, который последовательно создает файл, изменяет его содержимое и затем удаляет его.
`file_management.yml:`

<img width="565" height="269" alt="5_file_0" src="https://github.com/user-attachments/assets/e4cd4d01-dd46-4e56-aaea-966ff2069d3f" />

Запуск плейбука управления файлом:

`ansible-playbook file_management.yml`

<img width="905" height="759" alt="5_file_0_result" src="https://github.com/user-attachments/assets/bdec8694-5d14-483a-9a36-19a3198ba047" />

