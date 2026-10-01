# База кампании «Наэнтар» — сайт

Статический сайт нашей второй D&D-кампании (мир Наэнтар), собранный из Obsidian-волта движком [Quartz v5](https://quartz.jzhao.xyz/) и публикуемый на GitHub Pages. Устроен так же, как сайт первой кампании (`D:\Agent\dnd-wiki`), только без интерактивной карты.

**Адрес сайта:** https://danilshekarev.github.io/naentar-wiki/

## Как это устроено

- **Источник заметок** — Obsidian-волт `D:\Agent\Naentar-Vault` (его правишь в Obsidian).
- **Этот репозиторий** — движок Quartz и копия заметок в папке `content/`.
- При каждом `push` в ветку `main` GitHub Action собирает сайт и публикует его.

Волт и сайт — два отдельных репозитория. Заметки переносятся в `content/` скриптом синхронизации; вручную в `content/` ничего менять не нужно.

## Обновление сайта после правок в Obsidian

```powershell
# 1. Перенести свежие заметки из волта в content/
.\sync-content.ps1

# 2. Закоммитить и запушить — деплой запустится сам
git add -A
git commit -m "Обновление заметок"
git push
```

Скрипт `sync-content.ps1` зеркалит `Naentar-Vault` → `content/` (без `.git` и `.obsidian`), делает дашборд `00 Старт.md` главной страницей (`index.md`) и дописывает в конец каждой заметки сворачиваемую ссылку «📜 История заметки» на её коммиты в GitHub. Текст этой ссылки лежит в `footer-template.md`, чтобы сам скрипт оставался ASCII и без проблем разбирался в Windows PowerShell 5.1.

## Локальный предпросмотр

```powershell
npm ci                                            # один раз — зависимости
node ./quartz/bootstrap-cli.mjs plugin install    # один раз — плагины Quartz
node ./quartz/bootstrap-cli.mjs build --serve --port 8081 --wsPort 3002
```

Сайт откроется на http://localhost:8081. Порты 8081 и 3002 выбраны, чтобы предпросмотр не конфликтовал с сайтом первой кампании (у того 8080 и 3001), если запущены оба.

> На Windows вызывай CLI через `node ./quartz/bootstrap-cli.mjs ...` — `npx quartz ...` иногда сбоит с кэшем npx.

## Адрес без своего домена

Сайт живёт на `github.io` в подпапке репозитория, поэтому в `quartz.config.yaml` стоит `baseUrl: danilshekarev.github.io/naentar-wiki` — **с путём**. Из него Quartz берёт префикс для ссылок, а при локальном `--serve` префикс отключается сам. Плагин `cname` выключен: файл CNAME нужен только своему домену.

Если когда-нибудь понадобится свой домен: `baseUrl` становится голым хостом без пути, `cname` включается, в DNS добавляется CNAME на `danilshekarev.github.io`, а сам домен прописывается в **Settings → Pages → Custom domain**.

## Настройка

- `quartz.config.yaml` — заголовок, `baseUrl`, локаль (`ru-RU`), тема, плагины.
- `.github/workflows/deploy.yml` — сборка и деплой на GitHub Pages.

## Форк плагина графа (`plugins/graph`)

Граф — **локальная копия** `quartz-community/graph` с двумя добавленными опциями:

- **`hideTags: [...]`** — заметки с указанными тегами не попадают в граф, но остаются в поиске, RSS и sitemap (в upstream можно скрыть только узлы-теги, а не сами заметки).
- **`hideFolderPages: true`** — убирает сгенерированные страницы-индексы папок (`мир/index`, `боги/index`, …). Главная (`index`) не затрагивается.

Плюс исправлено поведение upstream: при `showTags: false` теперь убираются и **страницы тегов** (`tags/*`). Раньше они оставались в графе одинокими узлами, потому что живут в `contentIndex` как обычные страницы и на них никто не ссылается.

Настраивается в `quartz.config.yaml`:

```yaml
- source: ./plugins/graph          # локальный форк, не github:
  options:
    globalGraph:
      hideTags: [changelog, дашборд, служебное, сессия]
```

**Если правишь `plugins/graph/src/`** — пересобери `dist` и закоммить его:

```powershell
cd plugins/graph
npm install       # один раз
npm run build     # tsup -> dist/
```

`dist/` намеренно лежит в git (так же поставляются все upstream-плагины Quartz) — тогда CI берёт готовую сборку и не ставит зависимости плагина.

> ⚠️ Тонкости, на которые легко напороться:
> - `npx quartz plugin install` (режим lockfile) **молча пропускает** локальные плагины — поэтому в workflow есть отдельный шаг с `--from-config`.
> - Он же сначала делает `rm -rf .quartz/plugins/graph`: восстановленный из кэша старый upstream-граф заставил бы `--from-config` решить, что плагин уже стоит.
> - Записи `graph` в `quartz.lock.json` быть не должно. `--from-config` дописывает её с абсолютным путём этой машины; в CI такой путь не существует и install ругается «1 failed» (не фатально, но лучше запись удалить перед коммитом).
> - У `quartz/bootstrap-cli.mjs` в git должен стоять бит исполнения (`git update-index --chmod=+x quartz/bootstrap-cli.mjs`), иначе `npx quartz` в CI падает с exit 127 «Permission denied».

## Первичная настройка GitHub Pages

1. Создать на GitHub публичный репозиторий `naentar-wiki` и запушить `main`: `gh repo create naentar-wiki --public --source . --push`.
2. Включить Pages со сборкой из Actions: **Settings → Pages → Source → GitHub Actions** (то же самое командой: `gh api -X POST repos/DanilShekarev/naentar-wiki/pages -f build_type=workflow`).
