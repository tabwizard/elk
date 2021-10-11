# Описание playbook

## Общая часть

Playbook устанавливает и настраивает elasticsearch на хосты в группе `elasticsearch`, устанавливает и настраивает kibana на хосты в группе `kibana` и filebeat на хосты в группе `application`. Все настройки производятся для совместной работы.

## Описательная часть

```yaml
---
- name: Install Elasticsearch # Play для установки и настройки Elasticsearch
  hosts: elasticsearch # на группе хостов elasticsearch
  handlers: # опишем хэндлеры
    - name: restart Elasticsearch # хэндлер для рестарта процесса elasticsearch
      become: true # повышаем права
      systemd: # используем systemd
        name: elasticsearch # для процесса elasticsearch
        state: restarted # выполняем рестарт
        enabled: true # и заодно устанавливаем автозапуск при загрузке
  tasks: # таски
    - name: "Download Elasticsearch's rpm" # скачиваем rpm для elasticsearch
      get_url: # используем get_url для получения файла
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elk_stack_version }}-x86_64.rpm" # откуда
        dest: "/tmp/elasticsearch-{{ elk_stack_vesion }}-x86_64.rpm" # куда
      register: download_elastic # регистрируем результат выполнения
      until: download_elastic is succeeded # повторяем пока не получится, но не более 3-х раз
    - name: Install Elasticsearch # устанавливаем elasticsearch
      become: true # повышаем права
      yum: # используем yum для установки из rpm
        name: "/tmp/elasticsearch-{{ elk_stack_version }}-x86_64.rpm" # имя rpm-пакета
        state: present # будем устанаваливать
      notify: restart Elasticsearch # вызываем хэндлер для рестарта процесса elasticsearch
    - name: Configure Elasticsearch # настраиваем elasticsearch
      become: true # повышаем права
      template: # используем шаблон
        src: elasticsearch.yml.j2 # откуда
        dest: /etc/elasticsearch/elasticsearch.yml # куда
        mode: 0640 # устанавливаем права на файл
      notify: restart Elasticsearch # вызываем хэндлер для рестарта процесса elasticsearch

- name: Install Kibana # Play для установки и настройки Kibana
  hosts: kibana # на хосты в группе kibana
  handlers: # опишем хэндлеры
    - name: restart Kibana # хэндлер для рестарта процесса kibana
      become: true # повышаем права
      systemd: # используем systemd
        name: kibana # для процесса kibana
        state: restarted # выполняем рестарт
        enabled: true # и заодно устанавливаем автозапуск при загрузке
  tasks: # таски
    - name: "Download Kibana's rpm" # скачиваем rpm для kibana
      get_url: # используем get_url для получения файла
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ elk_stack_version }}-x86_64.rpm" # откуда
        dest: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm" # куда
      register: download_kibana # регистрируем результат выполнения
      until: download_kibana is succeeded # повторяем пока не получится, но не более 3-х раз
    - name: Install Kibana # устанавливаем kibana
      become: true # повышаем права
      yum: # используем yum для установки из rpm
        name: "/tmp/kibana-{{ elk_stack_version }}-x86_64.rpm" # имя rpm-пакета
        state: present # будем устанаваливать
      notify: restart Kibana # вызываем хэндлер для рестарта процесса kibana
    - name: Configure Kibana # настраиваем kibana
      become: true # повышаем права
      template: # используем шаблон
        src: kibana.yml.j2 # откуда
        dest: /etc/kibana/kibana.yml #куда
        mode: 0640 # устанавливаем права на файл
      notify: restart Kibana # вызываем хэндлер для рестарта процесса kibana

- name: Install Filebeat # Play для установки и настройки filebeat
  hosts: application # на хосты в группе application
  handlers: # опишем хэндлеры
    - name: restart Filebeat # хэндлер для рестарта процесса filebeat
      become: true # повышаем права
      systemd: # используем systemd
        name: filebeat # для процесса filebeat
        state: restarted # выполняем рестарт
        enabled: true # и заодно устанавливаем автозапуск при загрузке
  tasks: # таски
    - name: "Download Filebeat's rpm" # скачиваем rpm для filebeat
      get_url: # используем get_url для получения файла
        url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ elk_stack_version }}-x86_64.rpm" # откуда
        dest: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm" # куда
      register: download_filebeat # регистрируем результат выполнения
      until: download_filebeat is succeeded # повторяем пока не получится, но не более 3-х раз
    - name: Install Filebeat # устанавливаем filebeat
      become: true # повышаем права
      yum: # используем yum для установки из rpm
        name: "/tmp/filebeat-{{ elk_stack_version }}-x86_64.rpm" # имя rpm-пакета
        state: present # будем устанаваливать
      notify: restart Filebeat # вызываем хэндлер для рестарта процесса filebeat
    - name: Configure Filebeat # настраиваем filebeat
      become: true # повышаем права
      template: # используем шаблон
        src: filebeat.yml.j2 # откуда
        dest: /etc/filebeat/filebeat.yml # куда
        mode: 0640 # устанавливаем права на файл
      notify: restart Filebeat # вызываем хэндлер для рестарта процесса filebeat
    - name: Set filebeat systemwork # подключаем модули сбора системных логов
      become: true # повышаем права
      command: # выполняем команду в командной строке
        cmd: filebeat modules enable system # текст команды
        chdir: /usr/share/filebeat/bin # каталог выполнения
      register: filebeat_modules # регистрируем результат выполнения
      changed_when: filebeat_modules.stdout != 'Module system is already enabled' # изменения будут засчитаны только когда результат выполнения команды не равен 'Module system is already enabled' т.е. когда модуль установлен в первый раз
    - name: Load Kibana dashboard # загружаем дашборды в кибану
      become: true # повышаем права
      command: # выполняем команду в командной строке
        cmd: filebeat setup # текст команды
        chdir: /usr/share/filebeat/bin # каталог выполнения
      register: filebeat_setup # регистрируем результат выполнения
      until: filebeat_setup is succeeded # повторяем пока не получится, но не более 3-х раз
      notify: restart Filebeat # вызываем хэндлер для рестарта процесса filebeat
      changed_when: false # изменения не будут засчитаны для идемпотентности потому что результатом команды всегда будет `changed`
```
