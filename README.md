# Домашнее задание к занятию "ELK" - Наталья Мартынова (Пономарева)

### Задание 1.

Статус сервиса elastic: 
![Снимок1](https://github.com/NatoshFehn/hw-sdb-03/blob/main/Снимок1.png)   

Результат health check команды: 
![Снимок2](https://github.com/NatoshFehn/hw-sdb-03/blob/main/Снимок2.png)    

---

### Задание 2.

![Снимок3](https://github.com/NatoshFehn/hw-sdb-03/blob/main/Снимок3.png)  

---

### Задание 3.

Установила NGINX, изменила конфигурацию, пишем логи в syslog по порту 5445:  

```
events {
        worker_connections 1024;
}
 
http {
        include /etc/nginx/mime.types;
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/default;
        default_type application/octet-stream;

        log_format json escape=json
               '{'
                       '"Authorization":"$http_authorization",'
                       '"RequestTime":"$time_iso8601",'
                       '"RemoteAddress":"$remote_addr",'
                       '"RemotePort":"$remote_port",'
                       '"RemoteUser":"$remote_user",'
                       '"RequestHost":"$host",'
                       '"RequestPort":"$server_port",'
                       '"RequestMethod":"$request_method",'
                       '"RequestPath":"$request_uri",'
                       '"RequestBody":"$request_body",'
                       '"ResponseStatus":"$status",'
                       '"Upstream":"$upstream_addr",'
                       '"UpstreamPath":"$uri",'
                       '"UpstreamResponseTime":"$upstream_response_time"'
               '}';
 
       access_log syslog:server=0.0.0.0:5445 json;
```   
 
Конфигурация logstash настроена в /etc/logstash/conf.d/nginx-logstash.conf   
Читаем из 5445, парсим json, записываем в elasticsearch:

```
input {
  syslog {
    port => 5445
    tags => "nginx"
  }
}
 
filter {
    json{
        source => "message"
    }
    mutate {
        remove_field => ["message","timestamp","RequestTime","facility",> "facility_label","severity","severity_label","priority"]
    }
}
 
output {
  elasticsearch {
      hosts => ["https://localhost:9200"]
      index => "nginx-%{+yyyy.MM.dd}"
      user => "logstash_internal"
      password => "123456"
      cacert => "/etc/logstash/ca.crt"
  }
}
```  
 
Добавила пользователя logstash_internal, настроила роли и привелегии. Добавила Data  View:  
![Снимок4](https://github.com/NatoshFehn/hw-sdb-03/blob/main/Снимок4.png) 
 
Логи NGINX в интерфейсе Kibana
![Снимок5](https://github.com/NatoshFehn/hw-sdb-03/blob/main/Снимок5.png) 

---

### Задание 4.

В концигурации logstash для nginx добавляем input, будем просто помечать другим тегом:
```
input {
  beats {
    port => 5678
    tags => "nginx-from-filebeat"
  }
}
```
 
Конфигурация Filebeat:

```yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/nginx/*.log
output.logstash:
  # The Logstash hosts
  hosts: ["0.0.0.0:5678"]
```
 
Настройки логирования NGINX, попробуем теперь брать логи из файлов:
```
        access_log syslog:server=0.0.0.0:5445 json;
        access_log /var/log/nginx/access.log json;
        error_log /var/log/nginx/error.log;
```
Логи NGINX в интерфейсе Kibana с тегом nginx-from-filebeat
![Снимок6](https://github.com/NatoshFehn/hw-sdb-02/blob/main/Снимок6.png) 

---
