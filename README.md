# cad_dhcp

Конфигурация Kea DHCPv4 для выдачи адресов виртуальным машинам в сетях Proxmox SDN.

## Назначение

Сабмодуль хранит готовый файл:

```text
kea-dhcp4.conf
```

Он нужен для автоматической выдачи IP-адресов новым worker-ВМ и другим сервисным виртуальным машинам. Это уменьшает объём ручной настройки при масштабировании кластера.

## Что настроено

DHCP-сервер слушает интерфейсы:

```text
eth0
eth1
```

Если на вашей ВМ DHCP-сервера интерфейсы называются иначе, например `ens18` или `enp6s18`, их нужно заменить в блоке:

```json
"interfaces-config": {
  "interfaces": [ "eth0", "eth1" ]
}
```

## Подсети

| ID | Subnet | Pool | Gateway | DNS | Domain |
|---:|---|---|---|---|---|
| `1` | `10.10.0.0/24` | `10.10.0.100 - 10.10.0.199` | `10.10.0.1` | `10.10.0.2` | `cad.internal` |
| `2` | `10.10.1.0/24` | `10.10.1.100 - 10.10.1.199` | `10.10.1.1` | `10.10.0.2` | `cad.internal` |

Файл lease-базы:

```text
/var/lib/kea/kea-leases4.csv
```

Время аренды:

| Параметр | Значение |
|---|---:|
| `valid-lifetime` | `3600` секунд |
| `renew-timer` | `900` секунд |
| `rebind-timer` | `1800` секунд |

Лог:

```text
/var/log/kea-dhcp4.log
```

## Установка Kea DHCP

На Debian/Ubuntu:

```bash
apt update
apt install kea-dhcp4-server
```

Скопировать конфигурацию:

```bash
cp kea-dhcp4.conf /etc/kea/kea-dhcp4.conf
```

Проверить синтаксис:

```bash
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Перезапустить сервис:

```bash
systemctl restart kea-dhcp4-server
systemctl status kea-dhcp4-server
```

## Проверка работы

Посмотреть логи:

```bash
journalctl -u kea-dhcp4-server -f
```

Проверить файл аренд:

```bash
cat /var/lib/kea/kea-leases4.csv
```

На клиентской ВМ:

```bash
ip addr
ip route
cat /etc/resolv.conf
```

Ожидаемо:

- ВМ получает адрес из одного из пулов;
- gateway соответствует своей подсети;
- DNS указывает на `10.10.0.2`;
- domain установлен как `cad.internal`.

## Связь с другими сабмодулями

DHCP выдаёт DNS-сервер:

```text
10.10.0.2
```

Этот адрес должен соответствовать адресу Unbound из сабмодуля `cad_dns`.

Для worker-ВМ, созданных Terraform, используется сеть:

```text
10.10.1.0/24
```

Если Terraform выдаёт статические адреса через cloud-init, DHCP всё равно полезен для сервисных ВМ и тестовых клиентов. Если все worker-ВМ переводятся на DHCP, нужно изменить Terraform-модуль или использовать простой модуль `modules/vm`, где уже задано `address = "dhcp"`.

## Что менять при адаптации

Обычно нужно заменить:

- список интерфейсов `eth0`, `eth1`;
- подсети и DHCP pools;
- адрес gateway;
- DNS-сервер;
- домен `cad.internal`;
- путь к lease-файлу, если используется нестандартная схема хранения.

После любого изменения:

```bash
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
systemctl restart kea-dhcp4-server
```
