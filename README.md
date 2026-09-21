# Python и веб-технологии

Материалы курса: Python, Telegram-боты на aiogram, базы данных, API, веб, Mini App и деплой.

Сайт курса: https://ibulanov.github.io/pywebcourse/

## Структура

```text
docs/                 страницы сайта (материалы ученика)
├── index.md          о курсе
├── program.md        программа
├── setup.md          как работать с материалами
├── lessons/          страницы уроков
└── files/            файлы к урокам для скачивания
zensical.toml         настройки сайта и меню
.github/workflows/    сборка и публикация на GitHub Pages
```

## Локальный просмотр

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows; на macOS и Linux: source .venv/bin/activate
pip install zensical
zensical serve
```

Сайт откроется на http://localhost:8000. Изменения в файлах видны сразу.

## Публикация

При каждом пуше в ветку `main` GitHub Actions собирает сайт и публикует его на GitHub Pages. Один раз нужно включить это в настройках репозитория: **Settings → Pages → Source: GitHub Actions**.
