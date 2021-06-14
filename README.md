# 4.3. Языки разметки JSON и YAML
1. Вижу ошибки 2 видов: синтаксические и смысловые.  
1.1. Синтаксические ошибки: неправильно оформлено ключ:значение поля ip для элемента списка elements с "name" : "second".  
Старое значение: "ip : 71.78.22.43  
Правильное значение: "ip" : "71.78.22.43"  
1.2. Смысловые ошибки:  
1.2.1. В значении info вывод идет со знаком табуляции в конце, если это было бы на рабочем примере, я бы подумал что что-то с форматированием.  
1.2.2. Значение ip, для элемента списка elements с "name" : "first" указано 7175. Такого ip быть не может, тут какая-то ошибка.  
  
JSON с исправленными синтаксическими ошибками:
```json
{ "info" : "Sample JSON output from our service\t",
    "elements" :[
        { "name" : "first",
        "type" : "server",
        "ip" : 7175 
        },
        { "name" : "second",
        "type" : "proxy",
        "ip" : "71.78.22.43"
        }
    ]
}
```
  
2. Доработанная версия скрипта. Конфигурация считывается с json файла, после работы скрипт записывает json и yaml файлы.
```python
#!/usr/bin/env python3

import os
import socket
import json
import yaml

# JSON файл для хранения хостов и их IP адресов, формат ключ/значение
conf_file_json="2_config.json"
conf_file_yaml="2_config.yml"

with open(conf_file_json) as json_file:
    conf = json.load(json_file)

# Перебор всех хостов из файла конфигурации и опредление их IP адресов
for host, ip in conf.items():
    new_ip=socket.gethostbyname(host)

    # Если IP адреса отличаются, выводится предупреждение и сохранение нового IP адреса в файле конфигурации
    if (ip != new_ip):
        print ('[ERROR] {} IP mismatch: {} {}'.format(host,ip,new_ip))
        conf[host]=new_ip

# Печать всех сопоставлений хоста и IP адреса
# Сделано в отдельном цикле чтобы отделить ошибки от полезной информации
for host, ip in conf.items():
    print('{} - {}'.format(host,ip))

# Пишем в файл, параметр ident дает нам человекочитаемый формат
with open(conf_file_json, "w") as json_file:
    json.dump(conf, json_file, indent=2)

# Пишем в файл, параметр ident дает нам человекочитаемый формат
with open(conf_file_yaml, "w") as yaml_file:
    yaml_file.write(yaml.dump(conf,explicit_start=True))
```
  
Вывод json файла:
```json
{
  "drive.google.com": "64.233.165.194",
  "mail.google.com": "64.233.162.83",
  "google.com": "64.233.162.138"
}
```
  
Вывод yaml файла:
```yaml
---
drive.google.com: 64.233.165.194
google.com: 64.233.162.138
mail.google.com: 64.233.162.83
```
  
3. Скрипт для доп задачи:
```python
#!/usr/bin/env python3

import os
import sys
import json
import yaml
import argparse

# Определяем входной параметр
parser = argparse.ArgumentParser(description='Check and fix JSON and YAML file')
parser.add_argument(
    'in_file',
    type=str,
    help='provide filename for check and fix'
    )
args = parser.parse_args()

# Узнаем имя файла и его расширение
file_name, file_extension = os.path.splitext(args.in_file)

# Если расширение файла не являются json, yaml или yml пишем что файл не подходит
if not file_extension in ['.json','.yaml','.yml']:
    print ('Not JSON or YAML extension')
    sys.exit(1)

with open(args.in_file,'r') as in_f:
    data_file = in_f.read()

# Не нашел ничего умнее чем определять содержимое файла по первым символам, json файл начинается на {, yaml файл начинается на ---
# Если ориентироваться на ошибки функций десереализации, то получается что-то не то. yaml функции *_load без ошибок грузят json файлы.
# Если в файле будет синтаксическая ошибка, то тем более не понятно как определять формат

# Если файл начинвается с { то будем считать это JSON
if data_file.strip()[0] == '{':
    print ('Look as json file')

    try:
        data = json.loads(data_file)
    except json.JSONDecodeError as ex:
# Выводим соощение о синтаксической ошибки и номер строки
        print('Syntax error at line',ex.lineno)
# Выводим сообщение об ошибке полностью
        print(ex)
        sys.exit(1)

# Записываем в формат YAML
    print('Write file',file_name+'.yml')
    with open(file_name+'.yml','w') as out_f:
        out_f.write(yaml.dump(data,explicit_start=True))

# Если файл начинвается с --- то будем считать это YAML
if data_file.strip()[0:3] == '---':
    print ('Look as yaml file')
    try:
        data = yaml.safe_load(data_file)
    except yaml.YAMLError as ex:
# Выводим соощение о синтаксической ошибки и номер строки
        print('Syntax error at line',ex.problem_mark.line+1)
# Выводим сообщение об ошибке полностью
        print(ex)
        sys.exit(1)

#Записываем в формат JSON
    print('Write file',file_name+'.json')
    with open(file_name+'.json','w') as out_f:
        json.dump(data,out_f,indent=2)
```
