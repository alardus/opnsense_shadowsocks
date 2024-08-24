# Трафик на OPNSense через Shadowsocks

## В чем сложность

В OPNSense можно использовать Shadowsocks, но чтобы завернуть в него весь трафик который пойдет через роутер, нужно поднять дополнительный интерфейс и уже его заворачивать в SOCKS.

Для этого есть пакет `tun2socks`.

Ниже перевод оригинальной статьи https://blog.ohmykreee.top/article/setup-tun2socks-in-opnsense/ с комментариями, дополнениями и упрощениями.

## Предварительные требования
Что должно быть уже сделано перед тем, как двигаться дальше:
- Установлен OPNSense;
- У клиентов подключенных к роутеру с OPNSense работает интернет;
- Где-то настроен и запущен Shadowsocks сервер, и у вас есть к нему все креды;
- На OPNSense установлен пакет `os-shadowsocks` и настроен локальный клиент до сервера.

Чтобы проверить что все работает, при настройке локального клиента впишите в качестве адреса локального сервера IP адрес роутера (а не 127.0.0.1) и с другой машины выполните команду:

```
curl ip4.me/api/ --socks5 ROUTER_IP_ADDRESS:1080
```

В ответ должен вернуться внешний адрес SOCKS сервера. 

Если все хорошо, поменяйте в настройках клиента на OPNSense адрес локального сервера обратно на 127.0.0.1 и едем дальше.

# Устанавливаем tun2socks 
Идем в репозиторий проекта `https://github.com/xjasonlyu/tun2socks/releases` и забираем релиз для FreeBSD.

Логинимся на OPNSense через ssh (включите в настройках, по умолчанию он выключен), создаем папку для tun2socks:

```
mkdir /usr/local/tun2socks
```

и копируем в нее то, что скачали.


## Создаем конфигурационный файл

В этой же папке создаем конфиг `config.yaml`:
```
vi /usr/local/tun2socks/config.yaml
```

И пишем в него следующее (подставьте свои данные в поле с адресом):


```
# debug / info / warning / error / silent
loglevel: info

# URL format: [protocol://]host[:port]
proxy: socks5://ROUTER_IP_ADDRESS:1080

# URL format: [driver://]name
device: tun://proxytun2socks0

# Maximum transmission unit for each packet
mtu: 1500

# Timeout for each UDP session, default value: 60 seconds
udp-timeout: 120s
```

Сохраняйте и проверяйте:

```
/usr/local/tun2socks/tun2socks -config ./config.yaml
```

Должно запуститься и написать что-то в духе `[INFO]...`

Если все так, жмем Ctrl+C и едем дальше.

## Делаем из этого системный сервис
Создаем новый файл вот тут:

```
vi /usr/local/etc/rc.d/tun2socks
```

И пишем в него следующее:

```
#!/bin/sh

# PROVIDE: tun2socks
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="tun2socks"
rcvar="tun2socks_enable"

load_rc_config $name

: ${tun2socks_enable:=no}
: ${tun2socks_config:="/usr/local/tun2socks/config.yaml"}

pidfile="/var/run/${name}.pid"
command="/usr/local/tun2socks/tun2socks"
command_args="-config ${tun2socks_config} > /dev/null 2>&1 & echo \$! > ${pidfile}"

start_cmd="${name}_start"

tun2socks_start()
{
    if [ ! -f ${tun2socks_config} ]; then
        echo "${tun2socks_config} not found."
        exit 1
    fi
    echo "Starting ${name}."
    /bin/sh -c "${command} ${command_args}"
}

run_rc_command "$1"
```

Сохраняем и даем права на запуск:

```
chmod +x /usr/local/etc/rc.d/tun2socks
```

Теперь создаем:

```
vi /etc/rc.conf
```

и пишем в него:

```
tun2socks_enable="YES"
```

Сохраняем.

## Создаем файл для управления сервисом
Создаем новый файл вот тут:

```
vi /usr/local/opnsense/service/conf/actions.d/actions_tun2socks.conf
```

И пишем в него следующее:

```
[start]
command:/usr/local/etc/rc.d/tun2socks start
parameters:
type:script
message:starting tun2socks

[stop]
command:/usr/local/etc/rc.d/tun2socks stop
parameters:
type:script
message:stopping tun2socks

[restart]
command:/usr/local/etc/rc.d/tun2socks restart
parameters:
type:script
message:restarting tun2socks

[status]
command:/usr/local/etc/rc.d/tun2socks status; exit 0
parameters:
type:script_output
message:request tun2socks status
```

Сохраняем, перезапускаем `configd` чтобы применить изменения:

```
service configd restart
```


## Создаем новый плагин для OPNSense

Создаем новый файл вот тут:

```
vi /usr/local/etc/inc/plugins.inc.d/tuntosocks.inc
```

И пишем в него следующее:

```
<?php

/*
 * Copyright (C) 2017 EURO-LOG AG
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are met:
 *
 * 1. Redistributions of source code must retain the above copyright notice,
 *    this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED ``AS IS'' AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY
 * AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
 * AUTHOR BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY,
 * OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 * POSSIBILITY OF SUCH DAMAGE.
 */

/**
 * register service
 * @return array
 */
function tuntosocks_services()
{
    global $config;

    $services = array();
    $services[] = array(
        'description' => gettext('tun2socks gVisor TCP/IP stack'),
        'configd' => array(
            'restart' => array('tun2socks restart'),
            'start' => array('tun2socks start'),
            'stop' => array('tun2socks stop'),
        ),
        'name' => 'tun-socks',
        'pidfile' => '/var/run/tun2socks.pid'
    );
    return $services;
}

function tuntosocks_syslog()
{
    $logfacilities = array();
    $logfacilities['tun2socks'] = array(
        'facility' => array('tun2socks'),
    );
    return $logfacilities;
}
```

Сохраняем, инициализируем новый сервис:

```
pluginctl -s
```

В появившемся списке должна быть строка `tun-socks`, а в списке сервисов на OPNSense должен появиться соответствующий сервис. Внимание, речь не про раздел Services в левом меню, который у вас появился после установки пакета `os-shadowsocks`, и а именно про список системных сервисов (можете вывести их себе в виде списка через апплет на дашборде).

Если все хорошо, едем дальше.

## Запускаем сервис при старте системы
Создаем новый файл вот тут (номер, в данном случае 60, должен быть уникальным и не пересекаться с другими номерами сервисов):

```
vi /usr/local/etc/rc.syshook.d/early/60-tun2socks
```

И пишем в него следующее:

```
#!/bin/sh

# Start tun2socks service
/usr/local/etc/rc.d/tun2socks start
```

Сохраняем и даем права на запуск:

```
chmod +x /usr/local/etc/rc.syshook.d/early/60-tun2socks
```

Перезапускаем роутер и проверяем что `tun2socks` (в списке сервисов) запустился.


## Создаем новый порт и настраиваем шлюз

В разделе `Interfaces ‣ Assignments`, создайте новый порт для только что добавленного TUN устройства и сохраните настройки. 

В разделе `Interfaces ‣ [ТОЛЬКО_ЧТО_СОЗДАННЫЙ_ПОРТ]`, пропишите следующие настройки и сохраните:

| Setting | Value |
| ------------- | ------------- |
| Enable | Enable Interface |
| Description | TUN2SOCKS |
| IPv4 Configuration Type | Static IPv4 |
| IPv4 address | 10.0.3.1/24 |

Адрес IPv4, который здесь указываете, не должен пересекаться с другими используемыми подсетями.

В разделе `System ‣ Gateways`, добавьте новый шлюз со следующими параметрами:

| Setting | Value |
| ------------- | ------------- |
| Name | TUN2SOCKS_PROXY |
| Interface | TUN2SOCKS |
| Address Family  | IPv4 |
| IP address | 10.0.3.2 |
| Disable Gateway Monitoring | True |

Сохраняйте и нажимайте "Apply".

## Настраиваем алиасы для правил, которые далее будут использовать в фаерволле
В разделе `Firewall ‣ Aliases` создаем два новых алиаса:

| Alias Name  | Type | Data | 
| ------------- | ------------- | ------------- |
| NoProxyGroup | Network | `YOUR_LAN_NETWORK` |
| ProxyPort | Port(s) | 80, 443 |

В первом алиасе, если нужно, мы определяем подсеть которую не будем проксировать. Во втором -- определяем порты которые будем проксировать, в данном случае 80 и 443. 

Если хочется большего, вот тут хороший мануал объясняющий что еще можно вписать в алиасы и как: https://homenetworkguy.com/how-to/write-better-firewall-rules-opnsense-using-aliases/

## Конфигурируем фаерволл
В разделе `Firewall ‣ Rules ‣ LAN` нужно сделать несколько вещей:

Добавить новое правильно со следующими значениями:

| Setting | Value |
| ------------- | ------------- |
| TCP/IP Version | IPv4 |
| Protocol | TCP/UDP |
| Destination Invert | True |
| Destination | NoProxyGroup |
| Destination port range | ProxyPort to ProxyPort |
| Gateway | TUN2SOCKS_PROXY |

Сохраняем.

И двигаем это новое правило на самый верх, до `Default allow LAN` и всех остальных правил.

Сохраняем и нажимаем "Apply".

# Проверяем что все получилось
Если все получилось, то трафик клиентов подключенных к OPNSense теперь будет ходить через SOCKS. Чтобы проверить, сходите с клиента на https://ipinfo.io -- вместо вашего внешнего IP теперь должен определяться адрес прокси-сервера.

