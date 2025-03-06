## 1.1 Khởi tạo database SQL

```sh
$ mysql > CREATE DATABASE senlin CHARACTER SET utf8;
$ mysql > GRANT ALL PRIVILEGES ON senlin.* TO 'senlin'@'localhost' IDENTIFIED BY '<senlin_database_password>';
$ mysql > GRANT ALL PRIVILEGES ON senlin.* TO 'senlin'@'%' IDENTIFIED BY '<senlin_database_password>';
```

## 1.2 Khởi tạo thông tin project senlin

### Tạo user và gán quyền cho project service

```sh
$ openstack user create --password-prompt senlin
User Password:  <senlin_keystone_password>
Repeat User Password: <senlin_keystone_password>
$ openstack role add --project service --user senlin admin
```

### Tạo service senlin và endpoint

```sh
$ openstack service create --name senlin --description "senlin high availability" instance-ha
$ openstack endpoint create --region <Region> senlin public https://<domain>:8778
$ openstack endpoint create --region <Region> senlin internal http://<vip_internal>:8778
$ openstack endpoint create --region <Region> senlin admin http://<vip_internal>:8778
```

## 1.3 Khởi tạo config trên internal haproxy

Thêm file sau vào config haproxy `/etc/kolla/haproxy/services.d/senlin-api.cfg` và restart lại haproxy container:

```cfg
frontend senlin_api_front
    mode http
    http-request del-header X-Forwarded-Proto
    option httplog
    option forwardfor
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    bind <vip_internal>:8778
    default_backend senlin_api_back

backend senlin_api_back
    mode http
    server ctl1 <ip>:8778 check inter 2000 rise 2 fall 5
    server ctl2 <ip>:8778 check inter 2000 rise 2 fall 5
    server ctl3 <ip>:8778 check inter 2000 rise 2 fall 5

frontend senlin_api_external_front
    mode http
    http-request del-header X-Forwarded-Proto
    option httplog
    option forwardfor
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    bind <vip_external>:8778 ssl crt /etc/haproxy/haproxy.pem
    default_backend senlin_api_external_back

backend senlin_api_external_back
    mode http
    server ctl1 <ip>:8778 check inter 2000 rise 2 fall 5
    server ctl2 <ip>:8778 check inter 2000 rise 2 fall 5
    server ctl3 <ip>:8778 check inter 2000 rise 2 fall 5
```

## 1.4 Sync DB

```sh
$ docker exec -it -u0 senlin_api bash
=> senlin-manage db sync
```

