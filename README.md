#################################################################################################################
!ВНИМАНИЕ! 
- Для запуска используется WSL (Windows Subsystem for Linux), т.е. vagrantfile настроен для запуска ВМ через WSL.
- Для установки Grafana шаблонов, на хосте где запускается vagrant с ansible, нужно установить коллекцию grafana:
    ansible-galaxy collection install community.grafana
- Для запуска node-exporter нужно установить коллекцию prometheus:
    ansible-galaxy collection install prometheus.prometheus
- Из-за работы через WSL потребуется плагин для vagrant:
    vagrant plugin install virtualbox_WSL2
##################################################################################################################

Таск 1 --- Создайте Vagrantfile для запуска виртуальной машины с операционной системой Ubuntu 20.04.
Шаг 1. Установка Vagrant
Установка в WSL (Windows Subsystem for Linux)
Устанавливаем по инструкции с сайта и добавляем переменные среды (в wsl требуется для доступа vagrant к windows).
Шаг 2. Создание Vagrantfile
Заходим на сайт https://app.vagrantup.com/ , пишем в строке поиска "Ubuntu 20.04". Нажимаем на нее и получаем настройки.
Из документации vagrant для создания новой виртуальной машины на основе образа ubuntu вводим команду: 
vagrant init ubuntu/focal64 
В результате создается Vagrantfile
Шаг 3. Тестовый запуск
Тестовый запуск ВМ был не удачен
не мог подключиться по ssh  (гугл указал что это проблема из-за wsl)
установил плагин:
vagrant plugin install virtualbox_WSL2
После установки плагина подключился успешно.


Таск 2 --- Используйте Ansible для провижионинга и настройки виртуальной машины.
Шаг 1.  Добавляем ansible в Vagrantfile
В Vagrantfile в блоке provision указываем что будем использовать Ansible для настройки ВМ.
Устанавливаем Docker и Docker-compose для запуска контейнеров grafana и prometheus.
Docker и Docker-compose - это популярные инструменты, значит уже есть в наличии. Через ansible-galaxy выбираем роль, критерии – количество скачиваний и поддерживаемая платформа ubuntu.
Выбираем:
https://galaxy.ansible.com/ui/standalone/roles/nickjj/docker/documentation/
Дополнительно потребовалось установить модули docker и docker-compose для python.


Таск 3 --- Настройте автозапуск node_exporter на виртуальной машине.
Установливаем коллекцию (как реккомендует документация node-exporter'a):
ansible-galaxy collection install prometheus.prometheus
Добавляем роль в playbook:
https://github.com/prometheus-community/ansible/tree/main/roles/node_exporter


Таск 4 --- Установите и настройте Grafana и Prometheus в контейнерах Docker с помощью Ansible и docker-compose.
Контейнер прометеуса чтобы связаться с node-exporter должен знать его ip.
Поэтому вводим переменную ip_vm (ip-адрес виртуальной машины) нужна чтобы передать ip виртуальной машины в контейнер прометеуса (т.к. прометеус не имеет маршрута к node-exporter). 
Для этого добавляем запись в docker-compose.yml в сервис prometheus:
    extra_hosts:
      - "node-exporter:{{ ip_vm }}"
Она добавит в /etc/hosts имя node-exporter (указанное в prometheus.yml) и ip адрес виртуальной машины (ip_vm).
IP будет проброшен в эту переменную через шаблон джинджа.


Таск 5 --- Настройте Prometheus для сбора метрик от node_exporter и добавьте его в конфигурацию Prometheus.
В файлк prometheus задаем статическую цель для экспортера:
    static_configs:
      - targets: ['node-exporter:9100']
Связываение имени node-exporter и ip ВМ описано в таске №4.

Таск 6 --- Добавьте автоматическое создание dashboard Node Exporter Full в Grafana для отображения основных метрик виртуальной машины.
Для установки Grafana шаблонов, на хосте где запускается vagrant с ansible, нужно установить коллекцию grafana:
    ansible-galaxy collection install community.grafana
Далее с помощью модуля community.grafana.grafana_dashboard устанавливаем dashboard node-exporter-full.
Кроме того нужно установить "Data sources", сделаем это в плейбуке с помощью curl:
url -X POST -H "Content-Type: application/json" -d '{"name":"Prometheus", "type":"prometheus", "url":"http://prometheus:9090", "access":"proxy", "isDefault":true}' -u admin:admin http://localhost:3000/api/datasources

Спасибо что дочитали, хорошего дня!