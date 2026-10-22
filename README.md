# ssb-claude-v6-patch — Claude и Anthropic через IPv6 для Secret-Sing-Box

Патч-скрипт для серверов [Secret-Sing-Box (ssb)](https://github.com/A-Zuro/Secret-Sing-Box):
направляет трафик **Claude / Anthropic** через **глобальный IPv6** сервера.

Аналог решения для Xray / 3X-UI (outbound `freedom` + `sendThrough` + routing rule)
из статьи «Gemini — решение для РФ через ipv6» (checkvpn.net), адаптированный под
sing-box 1.12+ и конфиги, которые генерирует ssb.

## Зачем это нужно

Сервисы Anthropic (в том числе Claude и Claude API) могут блокироваться, отвечать
капчей или отказывать при выходе с IPv4 адресов дата-центров, тогда как с IPv6
того же сервера всё работает. Скрипт добавляет в серверный конфиг sing-box отдельный
outbound, который соединяется с Anthropic с исходным адресом вашего IPv6, и
маршрутизирует на него домены Anthropic.

Схема клиента не меняется (клиент → прокси → сервер), меняется только выход
сервера для доменов Anthropic. **Клиентские конфиги ssb трогать не нужно.**

## Соответствие Xray → sing-box

| Xray / 3X-UI (статья) | sing-box (этот патч) |
|---|---|
| outbound `freedom` | outbound `direct` |
| `sendThrough: <IPv6>` | `inet6_bind_address: <IPv6>` |
| `settings.domainStrategy: "UseIPv6"` | `domain_resolver: { server, strategy: "ipv6_only" }` |
| rule `geosite:anthropic` | `rule_set: ["anthropic"]` (geosite-anthropic.srs) |
| rule `domain:claude.ai` + `domain:api.anthropic.com` | `domain: [...]` |

## Что делает скрипт

1. **Берёт глобальный IPv6 сервера** (приоритет — от ручного ввода к автоопределению):
   1. аргумент командной строки: `ssb-claude-v6-patch.sh apply 2001:db8:1::10`;
   2. переменная окружения `CLAUDE_IPV6=...`;
   3. **интерактивный запрос** — скрипт показывает все глобальные IPv6 сервера
      и просит ввести нужный (номер из списка или полный адрес);
   4. автоопределение `ip -6 route get 2001:4860:4860::8888` → `src`
      — только если запуск не из терминала (cron/ansible) или как значение по умолчанию в запросе.

   Автоопределение при нескольких адресах может выбрать не тот, поэтому
   **рекомендуется вводить IPv6 вручную**. Любой введённый адрес валидируется
   (только глобальный IPv6, допускается суффикс `/64` или `%zone` — они отбрасываются),
   неверный адрес отклоняется с повторным запросом.
2. **Делает бэкап** `/etc/sing-box/config.json` в `/root/bak-singbox/config.json.bak-<дата>`
   (хранит последние 10 копий).
3. **Вносит три изменения в конфиг** (идемпотентно — повторный запуск не плодит дубликаты):
   * rule-set `anthropic` → `geosite-anthropic.srs` (копирует из webroot ssb или докачивает с SagerNet);
   * outbound `claude-v6`: `direct` + `inet6_bind_address` + `domain_resolver`
     со стратегией `ipv6_only` (аналог `UseIPv6`);
   * два routing-правила (перед остальными outbound-правилами):
     * `rule_set: ["anthropic"]` → `claude-v6`
     * `domain: ["claude.ai", "anthropic.com", "api.anthropic.com"]` → `claude-v6`
4. **Проверяет конфиг** через `sing-box check` — при ошибке автоматически
   откатывает бэкап.
5. **Перезагружает sing-box** (`systemctl reload`, fallback `systemctl restart`).
6. **Прогоняет тесты связи с Claude/Anthropic по IPv6** (7 проверок, см. ниже).

## Что покрывается: Claude, Claude API и другие сервисы Anthropic

Домены Anthropic берутся **не вручную**, а из rule-set `anthropic` (`geosite-anthropic.srs`,
сборка [SagerNet/sing-geosite](https://github.com/SagerNet/sing-geosite) из
[domain-list-community](https://github.com/v2fly/domain-list-community)) — категория
`anthropic` содержит домены `anthropic.com`, `claude.ai`, `claude.com`, `clau.de`,
`claude.dev`, `anthropic.team` и другие, то есть покрывает **все поддомены**,
включая Claude и Claude API.

| Сервис / домен | Как покрывается |
|---|---|
| `claude.ai` (Claude web) | суффикс в rule-set `anthropic` + явное правило патча |
| `api.anthropic.com` (Claude API) | суффикс `anthropic.com` + явное правило патча |
| `anthropic.com` | суффикс в rule-set `anthropic` + явное правило патча |
| `claude.com`, `clau.de`, `claude.dev` | суффиксы в rule-set `anthropic` |
| `claudeusercontent.com` (ресурсы UI) | свой суффикс в rule-set `anthropic` |

**Важно:** правило работает для трафика, который приходит на сервер через
прокси-клиент ssb; в клиентских конфигах Anthropic должен быть направлен в прокси
(в ssb это правило `rule_set: anthropic` уже есть — как и в этом патче).

## Требования

* сервер с установленным [Secret-Sing-Box](https://github.com/A-Zuro/Secret-Sing-Box)
  (стандартный `/etc/sing-box/config.json`);
* **глобальный** IPv6 (не ULA/link-local) — без него скрипт завершится с ошибкой
  **до** каких-либо правок конфига;
* sing-box **1.12+** (в ssb по умолчанию новее), python3, iproute2, systemd.

## Установка

```bash
curl -fsSL -o /usr/local/sbin/ssb-claude-v6-patch.sh \
  https://raw.githubusercontent.com/Anton98567/ssb-claude-v6-patch/main/ssb-claude-v6-patch.sh
chmod 700 /usr/local/sbin/ssb-claude-v6-patch.sh
```

## Применение

```bash
ssb-claude-v6-patch.sh apply    # добавить или обновить патч (режим по умолчанию)
ssb-claude-v6-patch.sh status   # показать текущее состояние патча
ssb-claude-v6-patch.sh remove   # убрать outbound и правила, добавленные патчем
ssb-claude-v6-patch.sh test     # проверить связь с Claude по IPv6 (без правок конфига)
```

### Как указать IPv6

```bash
ssb-claude-v6-patch.sh apply 2001:db8::1      # аргументом (самый простой способ)
CLAUDE_IPV6=2001:db8::1 ssb-claude-v6-patch.sh apply
ssb-claude-v6-patch.sh apply                     # интерактивный запрос адреса
```

### Тесты связи с Claude/Anthropic по IPv6

`ssb-claude-v6-patch.sh test` (и автоматически в конце `apply`) прогоняет 7 проверок:

1. IPv6 назначен на интерфейсе сервера;
2. у `claude.ai` есть AAAA-запись (в `domain_resolver` стоит `ipv6_only`);
3. `https://claude.ai/` через IPv6 отвечает (2xx–4xx);
4. `https://api.anthropic.com/v1/messages` через IPv6 отвечает (2xx–4xx);
5. `https://anthropic.com/` через IPv6 отвечает (2xx–4xx);
6. исходящий адрес соединения равен вашему IPv6 (нет NAT/маскарада);
7. **сквозной тест** через сам outbound `claude-v6` sing-box (Clash API) —
   если `clash_api` в конфиге не включён, проверка помечается `[SKIP]`.

* Итог `OK=6/7 FAIL=0` = всё работает (SKIP допустим, если нет Clash API);
* при любом `[FAIL]` скрипт завершается с кодом 1.
* Автотест в конце `apply` отключается переменной `CLAUDE_SKIP_TEST=1`.
* Таймаут HTTP-проверок — `CLAUDE_TEST_TIMEOUT` (по умолчанию 15 секунд).

### Переменные окружения

| Переменная | Назначение | По умолчанию |
|---|---|---|
| `CLAUDE_IPV6` | исходный IPv6 (имеет приоритет над запросом и автоопределением) | интерактивный запрос |
| `CLAUDE_CONFIG` | путь к конфигу sing-box | `/etc/sing-box/config.json` |
| `CLAUDE_BACKUP_DIR` | каталог бэкапов | `/root/bak-singbox` |
| `CLAUDE_SERVICE` | имя systemd-юнита | `sing-box` |

## Итоговый фрагмент конфига

```json
"outbounds": [
  {
    "type": "direct",
    "tag": "claude-v6",
    "inet6_bind_address": "2001:db8:1::10",
    "domain_resolver": { "server": "dns-main", "strategy": "ipv6_only" }
  }
],
"route": {
  "rules": [
    { "rule_set": ["anthropic"], "outbound": "claude-v6" },
    { "domain": ["claude.ai", "anthropic.com", "api.anthropic.com"], "outbound": "claude-v6" }
  ],
  "rule_set": [
    { "tag": "anthropic", "type": "local", "format": "binary",
      "path": "/var/www/<RULESET_DIR>/geosite-anthropic.srs" }
  ]
}
```

Правила ставятся до остальных outbound-правил конфига ssb, поэтому соединения
к Anthropic гарантированно попадают в `claude-v6`.

## Проверка работы

**1. Лог sing-box** (`/var/lib/sing-box/box.log` или `journalctl -u sing-box`):

```text
INFO inbound/trojan[trojan-in]: [user] inbound connection to claude.ai:443
INFO outbound/direct[claude-v6]: outbound connection to claude.ai:443
```

Для не-Anthropic доменов вместо `claude-v6` будет обычный `direct` (или `warp`).

**2. Clash API** (если включён в конфиге):

```bash
SECRET=$(jq -r '.experimental.clash_api.secret' /etc/sing-box/config.json)
curl -s -H "Authorization: Bearer $SECRET" http://127.0.0.1:9090/proxies \
  | jq -r '.proxies | keys[]'
curl -s -H "Authorization: Bearer $SECRET" \
  "http://127.0.0.1:9090/proxies/claude-v6/delay?url=https://claude.ai/&timeout=8000"
```

**3. Исходный адрес соединений:**

```bash
ss -tnp | grep sing-box
```

TCP-соединения к Anthropic должны идти с локального адреса вашего глобального IPv6.

## Восстановление

```bash
ls -1t /root/bak-singbox/
cp -a /root/bak-singbox/config.json.bak-<дата> /etc/sing-box/config.json
systemctl reload sing-box
```

Либо `ssb-claude-v6-patch.sh remove` — уберёт только элементы, добавленные патчем.

## Совместимость и нюансы

* **После полной переустановки ssb** конфиг генерируется заново — просто повторите
  `ssb-claude-v6-patch.sh apply`. Управление пользователями через `ssb`/`sbmanager`
  конфиг не перетирает (правки точечные), патч сохраняется.
* Поле `domain_resolver.strategy` в sing-box 1.14 помечено как deprecated
  (план удаления — 1.16). На sing-box 1.12–1.15 работает; при переходе на 1.16+
  поле `strategy` в скрипте нужно будет убрать или заменить на новую механику
  выбора семейства адресов.
* Если глобальных IPv6 несколько — **не полагайтесь на автоопределение**: скрипт
  покажет список адресов и попросит выбрать нужный вручную (либо задайте его
  аргументом/`CLAUDE_IPV6`). В режиме без терминала (cron) берётся адрес
  исходящего маршрута (`ip -6 route get`).
* Скрипт идемпотентен: повторный `apply` сначала удаляет свои элементы
  (outbound с тегом `claude-v6`, его правила) и добавляет заново.
* Патч работает на стороне **сервера**: клиентские конфиги ssb уже отправляют
  Anthropic (`rule_set: anthropic`) в прокси, а сервер выбирает для него IPv6-выход.
