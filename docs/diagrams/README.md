# Диаграммы в SVG

Здесь лежат все диаграммы из документации в виде отдельных SVG. На странице широкие диаграммы уменьшаются под ширину колонки, а SVG открывается в натуральную величину, и его можно приближать без потери качества.

| Файл | Диаграмма |
|---|---|
| [01-context-map-1.svg](01-context-map-1.svg) | Карта контекстов |
| [02-c4-container-1.svg](02-c4-container-1.svg) | C4, уровень Context |
| [02-c4-container-2.svg](02-c4-container-2.svg) | C4, уровень Container |
| [04-main-process-1.svg](04-main-process-1.svg) | Состояния аренды |
| [04-main-process-2.svg](04-main-process-2.svg) | Успешный сценарий |
| [04-main-process-3.svg](04-main-process-3.svg) | Сбой 1. Машина не открылась |
| [04-main-process-4.svg](04-main-process-4.svg) | Сбой 2. Платёжный шлюз завис во время бронирования |
| [04-main-process-5.svg](04-main-process-5.svg) | Сбой 3. Оплата после поездки |

## Как обновить

Исходники диаграмм находятся в блоках `mermaid` внутри `.md`, SVG собираются из них. Номер в имени файла соответствует порядку диаграммы в документе. После правки диаграммы запустите из корня репозитория:

```bash
tmp=$(mktemp -d)
for f in 01-context-map 02-c4-container 04-main-process; do
  npx -p @mermaid-js/mermaid-cli mmdc -c diagrams/mermaid.json -i $f.md -o $tmp/$f.md -a diagrams -b white
done
```

`mermaid.json` отключает подгонку под ширину окна, чтобы SVG открывался в натуральную величину. Если `mmdc` пишет, что не нашёл браузер, один раз выполните `npx puppeteer browsers install chrome-headless-shell`.
