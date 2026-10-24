# meal-prep-rohlik: krabičkový jídelníček s daty z Rohlik.cz

Plugin pro Claude Cowork, který plánuje krabičkovou dietu (meal prep) s reálnými nutričními hodnotami a cenami produktů z Rohlik.cz. Spočítá makra pro jednu nebo víc osob, sestaví jídelníček na několik dní, dorovná porce na celá balení, aby zbylo co nejmíň odpadu, a připraví nákupní seznam, který na požádání vloží do košíku na Rohlíku. Autor: Filip Oborník. Aktuální verze je 0.3.0.

## Jak se používá

Plugin je konverzační, příkazy si pamatovat nemusíš. Stačí napsat třeba „nastav mi makra", „naplánuj jídelníček na týden" nebo „udělej nákup z posledního jídelníčku". Skill `meal-prep-orchestrator` pozná, o co jde, doplní chybějící kroky (bez profilu nejdřív založí profil, bez jídelníčku nabídne, že ho vytvoří) a spustí správný workflow. Plugin komunikuje česky.

Jediný slash command je `/cache-info`: přehled lokální cache produktů a receptů, refresh starých cen, mazání položek a export do CSV.

Hlavní pravidla pluginu:

- Nutriční data a ceny bere jen z Rohlik.cz přes MCP, nikdy z webu.
- Nejdřív se dívá do lokální cache: nutriční hodnoty v ní platí 30 dní, ceny 24 hodin.
- Maso, drůbež a ryby se v jídelníčku vždy spotřebují celé. S mrazákem se nepočítá.
- Hlídá cenový strop na den a osobu z nastavení. Když ho jídelníček překročí, navrhne levnější nebo zlevněné položky, limit sám nezvyšuje.
- Přidání do košíku, vytvoření shopping listu a jakékoli mazání dělá jen po výslovném potvrzení v chatu. Objednávku nikdy nedokončuje a platební údaje nezadává, checkout je vždy na uživateli.

## Co potřebuje

- **Claude Cowork** s nainstalovaným pluginem.
- **Účet na Rohlik.cz.** Plugin se připojuje k oficiálnímu Rohlík MCP serveru `rohlik-official` (HTTP, `https://mcp.rohlik.cz/mcp`, definovaný v `plugin-source/.mcp.json`). Přes něj hledá produkty, čte nutriční tabulky, slevy a recepty a pracuje s košíkem a shopping listy.
- **Proměnné prostředí** `ROHLIK_USERNAME` a `ROHLIK_PASSWORD` s přihlašovacími údaji k Rohlik.cz. V Cowork se nastavují v Settings → MCP servery → meal-prep-rohlik. Do repa ani do pluginu se nepíšou.
- Skilly pracují přes Bash (složky, atomické zápisy cache) a pro úpravu `CLAUDE.md` volají `python3`.

## Instalace a první spuštění

1. Nainstaluj `meal-prep-rohlik.plugin` do Claude Cowork.
2. Nastav `ROHLIK_USERNAME` a `ROHLIK_PASSWORD` (viz výše).
3. V pracovní složce, kde chceš jídelníčky mít, napiš „nastav mi makra" nebo „naplánuj jídelníček". Plugin tě provede založením profilu a globálního nastavení.

## Co plugin ukládá

Všechno jde do pracovní složky uživatele, ne do složky pluginu. Data tak zůstávají s projektem a může je sdílet celá domácnost:

```
.krabickova-dieta/
├── profiles/<jmeno>.json   profil a makra jedné osoby
├── settings.json           cenový strop, pantry, strategie odpadu
├── cache/products.json     nutriční hodnoty, balení a ceny z Rohlíku
├── cache/recipes.json      opakovaně použitelné recepty
└── jidelnicky/             jídelníčky a nákupní seznamy v Markdownu
CLAUDE.md                   managed sekce se stavem projektu
```

V `CLAUDE.md` si plugin udržuje sekci mezi značkami `<!-- meal-prep-rohlik:start -->` a `<!-- meal-prep-rohlik:end -->`: profily, nastavení, poslední jídelníček a stav cache. Díky ní Claude ví, kde projekt skončil, i v nové konverzaci. Obsahu mimo značky se plugin nedotýká.

## Struktura repa

```
meal-prep-rohlik.plugin          zabalený plugin k instalaci (ZIP)
plugin-source/                   zdrojové soubory pluginu
├── .claude-plugin/plugin.json   manifest: název, verze, popis, klíčová slova
├── .mcp.json                    připojení k Rohlík MCP
├── README.md                    podrobný popis, architektura a changelog
├── commands/cache-info.md       příkaz /cache-info
└── skills/                      skilly, každý ve vlastní složce se SKILL.md
```

| Skill | Typ | K čemu |
|---|---|---|
| `meal-prep-orchestrator` | workflow | vstupní bod: zjistí stav, rozpozná záměr, spustí workflow; šablona managed sekce `CLAUDE.md` |
| `setup-macro-profile` | workflow | založení, úprava a smazání profilu osoby (BMR, TDEE, makra) |
| `generate-meal-plan` | workflow | návrh jídelníčku, přesná makra z Rohlíku, optimalizace odpadu, cenová kontrola |
| `build-shopping-list` | workflow | nákupní seznam z hotového jídelníčku s aktuálními cenami, košík nebo shopping list |
| `rohlik-product-lookup` | znalostní | jak volat Rohlík MCP, schéma a pravidla cache |
| `waste-minimization` | znalostní | algoritmus dorovnání porcí na celá balení |
| `macro-balancing` | znalostní | vzorce BMR a TDEE, rozložení maker, tolerance, bezpečnostní limity |
| `meal-planning` | znalostní | zásady krabičkového vaření, šablony jídel, výchozí pantry a nastavení v `references/` |

Podrobnosti jsou v `plugin-source/README.md` a v jednotlivých `SKILL.md`.

## Úpravy a nové sestavení balíčku

Upravuje se jen `plugin-source/`. Soubor `meal-prep-rohlik.plugin` je ZIP, který má v kořeni přímo obsah `plugin-source/` bez nadřazené složky. Po změně ho přegeneruj, například takhle:

```bash
cd plugin-source
rm -f ../meal-prep-rohlik.plugin
zip -r ../meal-prep-rohlik.plugin . -x '*.DS_Store'
```

Verze pluginu je v `plugin-source/.claude-plugin/plugin.json`, changelog na konci `plugin-source/README.md`.
