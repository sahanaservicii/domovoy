---
name: openwrt-admin
description: "Безопасное администрирование OpenWrt по SSH: uci, ubus, fw4/nft, dnsmasq, Wi-Fi, VLAN, WireGuard, диагностика и восстановление."
---

# OpenWrt без интернета

Роутер: `192.168.1.1`, пользователь `root`, только SSH-ключ.
Все команды — через `homelab.py router`:

```bash
H=/root/workspace/homelab/homelab.py
python3 $H router status
python3 $H router exec "uci show network"
python3 $H router uci network
```

`router exec` выполняет строку на роутере. Опасные команды требуют `--yes`.

## Сначала факты

```bash
ubus call system board
ubus call system info
ubus call network.interface.wan status
ip -br a; ip r
cat /tmp/dhcp.leases
logread | tail -50
dmesg | tail -30
df -h / /tmp
free -m
```

## UCI: безопасный цикл

```bash
uci show <config>                 # 1. прочитать
cp /etc/config/<config> /root/<config>.bak-$(date +%s)
uci set <config>.<section>.<opt>='value'
uci commit <config>               # 2. зафиксировать
/etc/init.d/<service> reload      # 3. применить точечно
uci show <config>                 # 4. проверить
```

Не делай `uci commit` без проверки diff. Не перезагружай роутер, если хватает
reload. Перед правкой сети/firewall — `homelab.py router backup`.

## Типовые задачи

- **DHCP static lease**
  ```bash
  uci show dhcp
  uci add dhcp host
  uci set dhcp.@host[-1].name='tv'
  uci set dhcp.@host[-1].ip='192.168.1.139'
  uci set dhcp.@host[-1].mac='AA:BB:CC:DD:EE:FF'
  uci commit dhcp
  /etc/init.d/dnsmasq restart
  ```
- **DNS/dnsmasq**: `uci show dhcp`; локальные записи — `dhcp.@dnsmasq[0].address`,
  upstream — `.server`. Проверка: `nslookup host 192.168.1.1`,
  `logread -e dnsmasq | tail`.
- **Wi-Fi**: `iwinfo`; `uci show wireless`; `wifi reload`; не меняй канал вслепую.
  Для IoT — отдельный SSID/VLAN, WPA2/WPA3, без WPS.
- **Firewall**: `fw4 print` (не `iptables-save`), `uci show firewall`.
  Пробросы WAN — `firewall.@redirect`, правила — `firewall.@rule`.
  Никаких UPnP/DMZ. После `fw4 reload` проверь SSH и LAN.
- **WireGuard**: `uci show network | grep -i wg`, `wg show`; интерфейс обычно
  `network.wg0`. Не публикуй приватные ключи.
- **VLAN**: `uci show network` (device/bridge-vlan), `bridge vlan show`.
  Ошибка в VLAN легко отрезает Wi-Fi — делай по шагам, с консолью под рукой.
- **Маршруты/MTU**: `ip r`, `ip link`, `ping -c3 -s1472 1.1.1.1` (если ICMP
  разрешён), `traceroute` может отсутствовать.

## Диагностика

```bash
logread -f                       # живой лог
ubus list
ubus call dhcp.leases
cat /proc/net/nf_conntrack | head
nft list ruleset | head -80
iwinfo <iface> assoclist
```

Если SSH не отвечает: проверить питание/LAN, IP, ключ, `dropbear`; при полной
потере — failsafe (выключить питание, включить и быстро подключиться к
`192.168.1.1` по Ethernet, `failsafe`). Не делай этого без инструкции
пользователю.

## Ограничения

- `opkg update/install` не работают офлайн. Не предлагай их.
- Не меняй root-пароль/ключи без явного запроса.
- Не выводи PSK, ключи WireGuard, пароли.
- Один шаг — одна проверка. После каждого изменения фиксируй результат.
