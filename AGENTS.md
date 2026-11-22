# vpanel — контекст для AI-агента

## Быстрый старт

Пользователь общается по-русски. Рабочая папка Windows: `F:\vpanel`, shell — Git Bash. Не читать и не публиковать приватные ключи или пароли. Не создавать новые ключи и не удалять пользовательские конфиги без явной просьбы.

Публичный репозиторий: `https://github.com/mchemyakin/vpanel`.
Актуальная версия панели: `vpanel_v13`, файл `vpanel_v13.sh`, тег `vpanel_v13`.
Предыдущие версии не перезаписывать. Для новой разработки скопировать последнюю версию в `vpanel_v14.sh`, `vpanel_v15.sh` и т. д. После проверки создать commit и одноимённый тег, отправить `main` и тег.

Перед работой проверить живое состояние серверов: пользователь часто меняет клиентов, upstream, порты и DEFAULT после предыдущего диалога.

## Доступы

- `45.131.43.33`: SSH `root`, ключ `/f/vpanel/SelectelAR`.
- `81.163.30.100`: SSH `root`, исходный ключ `/f/vpanel/ssh ключ.txt` содержит CRLF и напрямую не работает.
- Для `81` создать временный ключ, использовать и обязательно удалить после работы:

```bash
sed 's/\r$//' '/f/vpanel/ssh ключ.txt' > '/f/vpanel/.ssh-key-normalized-temp'
chmod 600 '/f/vpanel/.ssh-key-normalized-temp'
ssh -i '/f/vpanel/.ssh-key-normalized-temp' root@81.163.30.100
rm -f '/f/vpanel/.ssh-key-normalized-temp'
```

- `103.112.68.10`: доступ по паролю был выдан пользователем в диалоге, не хранить его здесь. TCP/UDP от `45`, `81` и рабочего ПК ранее не доходили до `ens3`, хотя ping и локальные listeners работали. Сначала проверять сетевой путь, не менять панель вслепую.
- `5.180.97.161`: нужно развернуть панель, но на момент последней работы TCP 22 был недоступен. Изменений на нём не выполнено. Пароль есть только в исходном сообщении пользователя, не копировать сюда.

## Текущее состояние

На `45.131.43.33` и `81.163.30.100` установлена `vpanel_v13`. Панели:

- `https://45.131.43.33:8443`
- `https://81.163.30.100:8443`

Пароли существующих панелей сохранять. Не переустанавливать рабочую панель без необходимости.

Незавершённая задача: после открытия SSH на `5.180.97.161` установить `vpanel_v13`, создать клиента для `81.163.30.100`, добавить ключ как upstream на `81`, проверить протоколы и реальный HTTP. Для нового сервера выбрать отдельную базовую подсеть, например `11.30.0`, чтобы избежать конфликтов с существующими серверами.

## Архитектура

Один Bash-установщик содержит embedded Next.js приложение:

- `/opt/wg-web/app/app/lib/core.mjs` — основная логика;
- `/opt/wg-web/app/app/api/[[...path]]/route.js` — API;
- `/opt/wg-web/app/app/page.js` — UI;
- `/opt/wg-web/app/worker.js` — фоновые задачи;
- `/opt/dd-awg-bot/bot.py` — Telegram bot;
- `/opt/wg-web/docker-compose.yml` — app + Caddy;
- `/etc/wireguard/wg-web/awg-state.json` — HPK, адресный план и ingress-порты.

Контейнер работает с `network_mode: host`, `privileged`, `pid: host`, а host-команды выполняются через `nsenter`.

Ingress по умолчанию:

- WG: `wg-in`, UDP 52000;
- AWG 2.0: `awg-in`, UDP 55000;
- AWG 3.1: `awg3-in`, UDP 58000.

AWG 2.0 использует legacy-параметры без `HeaderProtectionKey`, обычно `S4=7` и диапазоны H1–H4. AWG 3.1 использует общий HPK из `awg-state.json`, `ContentPaddingAddition=32-128`, `S1–S4>=12`, `H1=1`, `H2=2`, `H3=3`, `H4=4`. HPK нельзя выводить в логи.

VLESS сервер: Xray Reality TCP 443 и xHTTP 8433. Состояние включения пользователь может менять.

Upstream:

- WG/AWG `.conf`: интерфейсы `awg-up-<id>`;
- `vless://`: отдельный Xray TUN, интерфейсы `vl-up-<id>`, сервисы `dd-vless-up-<id>.service`;
- поддерживаются VLESS Reality TCP и xHTTP.

Routing:

- upstream tables `61000 + slot`;
- upstream marks `0x100N0000`;
- client marks `0x10000000 + octet`;
- rules 17001+ для upstream, 18000+octet для клиентов;
- collision table 59990;
- collision priorities: 0 — upstream `iif ... lookup local`, 1 — client address в 59990, 2 — обычная local table;
- `accept_local` включать только для collision-интерфейса и только если sysctl-файл существует;
- iptables применяется через `iptables-restore --noflush`;
- `ip -batch` применяется через `command ip`, batch-файл создаётся заранее даже пустым.

Xray должен принудительно перезапускаться при изменении набора VLESS-клиентов. JSON без restart недостаточен: runtime иначе отклоняет новый UUID.

## Важные исправления по версиям

- v07: обработка конфликтов локальных и upstream IP;
- v08: VLESS upstream через Xray TUN;
- v09: корректный restart Xray при смене клиентов;
- v10: пользовательские WG/AWG2/AWG3 UDP-порты через установщик, пункт меню 3;
- v11: guard для отсутствующих ingress и правильные collision priorities;
- v12: безопасный пустой `ip.batch`;
- v13: отображение `Panel version: vpanel_v13` в Settings.

## Правила разработки

- Сначала воспроизвести проблему и найти причину, не менять firewall/порты вслепую.
- Сохранять совместимость WG, AWG2, AWG3, VLESS TCP/xHTTP, DIRECT и DEFAULT.
- При изменении порта перевыпускать только ключи затронутого протокола.
- TCP 443 Xray и UDP 443 AWG могут сосуществовать. WG/AWG2/AWG3 не могут делить один UDP-порт.
- Не копировать старый firewall целиком: рабочая старая панель использовала клиентские подсети `11.11/11.12` и upstream `10.7/10.8`, поэтому там не было IP collisions.
- Избегать одинакового IP у локального клиента и upstream. Для тестов не использовать loop `81 → 45 → 81`: он создаёт NAT/conntrack артефакты.
- Перед рискованными изменениями делать backup, если пользователь явно не запретил.
- Временные клиенты `zzmx`, `zzm2`, upstream `zx-*`, `diag*` и тестовые конфиги удалять, DEFAULT восстанавливать.
- После тестов удалять локальные `.link`, `.conf`, askpass и normalized key файлы.

## Проверки

Локально:

```bash
bash -n vpanel_v13.sh
```

Embedded JS извлекать из heredoc и проверять через `node --check`. Для генераторов конфигов делать unit-проверки с тестовыми ключами, не с реальными секретами.

На сервере:

```bash
docker exec wg-web-app-1 node --input-type=module -e \
  'const core = await import(`./app/lib/core.mjs`); console.log(await core.listClients());'

curl -k -sS -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8443/
```

Проверять реальные вызовы, не только компиляцию. Для upstream:

```bash
curl -sS --interface <awg-up-id-or-vl-up-id> --max-time 25 https://api.ipify.org
```

Ожидаемый IP — адрес удалённого VPN-сервера. Полная матрица: WG, AWG2, AWG3, VLESS TCP, xHTTP; DIRECT и DEFAULT; по возможности обе стороны. После неё выполнить cleanup.

## Деплой

Для изменений core/page загрузить соответствующий embedded файл в `/opt/wg-web/app/...`, новый installer в `/root/`, затем:

```bash
docker compose -f /opt/wg-web/docker-compose.yml up -d --build
```

Проверить сервисы, routing, handshake/HTTP и `HTTPS 200`. Не считать успешный exit installer достаточным: читать его лог.

Для repair существующей панели использовать неинтерактивный ввод осторожно и перенаправлять лог в файл. При EOF старые prompt-циклы могут бесконечно печатать `invalid selection`.

## Git и секреты

Перед commit:

```bash
git status --short
bash -n <version>.sh
git diff --cached --check
```

Сканировать staged files на SSH private keys и GitHub tokens. `SelectelAR`, `ssh ключ.txt`, `.env`, normalized keys, тестовые конфиги и backups не коммитить. Текущий `.gitignore` сохранять.
