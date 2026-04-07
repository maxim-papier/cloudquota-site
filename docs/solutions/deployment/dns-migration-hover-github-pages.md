---
title: "DNS-миграция cloudquota.com с SiteGround/Framer на GitHub Pages"
date: 2026-04-06
category: deployment
tags: [dns, github-pages, hover, https, domain-management]
component: cloudquota-site
severity: low
symptoms:
  - "Старые A-записи (SiteGround) направляли трафик на прежний хостинг"
  - "CNAME www указывал на sites.framer.app"
  - "GitHub Pages не был настроен как основной хостинг"
root_cause: "DNS-записи от предыдущих провайдеров (SiteGround, Framer) оставались активными после решения о переезде на GitHub Pages"
---

# DNS-миграция cloudquota.com на GitHub Pages

## Контекст

Домен cloudquota.com был зарегистрирован на Hover. DNS-записи указывали на старую инфраструктуру: A-записи на SiteGround, CNAME www на Framer. Параллельно на домене работала почта Microsoft 365 (Outlook). Нужно было перенести сайт на GitHub Pages, не сломав почту.

## Корневая причина

При переходе на GitHub Pages DNS-записи от предыдущих провайдеров не были обновлены. Старые A-записи конфликтовали бы с новыми, а CNAME www указывал на Framer вместо GitHub.

## Решение

### 1. Аудит существующих DNS-записей

Перед любыми изменениями — полная инвентаризация записей с разделением на "удалить" и "не трогать".

### 2. Удаление старых записей

| Тип | Host | Value | Причина удаления |
|-----|------|-------|------------------|
| A | * | 216.40.34.41 | Дефолтная заглушка Hover |
| A | @ | 31.43.161.6 | Старый хостинг SiteGround |
| A | @ | 31.43.160.6 | Старый хостинг SiteGround |
| CNAME | www | sites.framer.app | Старый сайт на Framer |

### 3. Добавление записей GitHub Pages

```
A     @    185.199.108.153
A     @    185.199.109.153
A     @    185.199.110.153
A     @    185.199.111.153
CNAME www  maximpapier.github.io
```

### 4. Сохранение почтовых записей (Microsoft 365)

Эти записи **не трогали** — они отвечают за email:

```
MX    @            cloudquota-com.mail.protection.outlook.com
TXT   @            MS=ms14621612
TXT   @            v=spf1 include:spf.protection.outlook.com -all
CNAME autodiscover autodiscover.outlook.com
CNAME mail         mail.hover.com.cust.hostedemail.com
```

### 5. Настройка GitHub Pages через API

```bash
# Привязка домена
gh api repos/maxim-papier/cloudquota-site/pages -X PUT --input - <<< '{"cname":"cloudquota.com"}'

# Включение HTTPS (после выпуска сертификата, ~15 мин)
gh api repos/maxim-papier/cloudquota-site/pages -X PUT --input - <<< '{"cname":"cloudquota.com","https_enforced":true}'
```

**Важно:** при попытке включить HTTPS сразу после привязки домена GitHub API возвращает ошибку `"The certificate does not exist yet"`. Нужно подождать 5-15 минут, пока Let's Encrypt выпустит сертификат.

## Верификация

```bash
# DNS-резолвинг — должны вернуться 4 IP GitHub
dig cloudquota.com +short
# 185.199.108.153, 185.199.109.153, 185.199.110.153, 185.199.111.153

# WWW — CNAME → GitHub → те же IP
dig www.cloudquota.com +short
# maximpapier.github.io → 4 IP

# HTTP-заголовки — Server: GitHub.com
curl -sI http://cloudquota.com | grep Server
# Server: GitHub.com
```

## Подводные камни

1. **HTTPS нельзя включить сразу** — GitHub Pages нужно время на выпуск SSL-сертификата через Let's Encrypt. Ждать 5-30 минут после привязки домена.
2. **Почтовые записи легко удалить случайно** — MX, SPF, DKIM, autodiscover выглядят как "лишние" записи, но их удаление сломает email. Всегда составлять список "не трогать" перед началом.
3. **Дефолтная A-запись Hover** (`* → 216.40.34.41`) — добавляется регистратором автоматически. Её нужно удалить, иначе wildcard будет конфликтовать.

## Чеклист для будущих DNS-миграций

### До начала
- [ ] Задокументировать все существующие DNS-записи
- [ ] Разделить на "удалить" / "сохранить" / "добавить"
- [ ] Убедиться, что новая инфраструктура работает (сайт доступен на username.github.io)

### Во время миграции
- [ ] Удалить старые A/CNAME записи
- [ ] Добавить новые записи
- [ ] **Не трогать** почтовые записи (MX, SPF, DKIM, autodiscover)
- [ ] Подождать пропагацию DNS (~15-30 мин)

### После миграции
- [ ] `dig domain.com +short` — проверить A-записи
- [ ] `curl -sI http://domain.com` — проверить Server header
- [ ] Включить HTTPS (после выпуска сертификата)
- [ ] Проверить работу почты (отправить/получить тестовое письмо)

## Связанные коммиты

- `cd9c1bc` — ci: add GitHub Pages deploy workflow
- `025f37e` — chore: add CNAME for cloudquota.com custom domain

## Ссылки

- [GitHub Pages: настройка кастомного домена](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site)
- [GitHub Pages API](https://docs.github.com/rest/pages/pages)
- Регистратор: [Hover](https://hover.com) → DNS → cloudquota.com
