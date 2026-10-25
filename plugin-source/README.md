# meal-prep-rohlik

Plugin pro Claude Cowork, který sestavuje **krabičkový jídelníček** s reálnými nutričními hodnotami a cenami z **Rohlik.cz**, hlídá makra více osob a chytře minimalizuje odpad surovin.

**Konverzační, ne příkazový.** Uživatel mluví přirozeně — *„naplánuj mi jídelníček na týden"*, *„kolik mám jíst bílkovin"*, *„udělej nákup z toho jídelníčku"* — a plugin sám pozná, co dělat. Slash command zůstává jen jeden, na věc, co se dá těžko popsat slovy (správa cache).

## Architektura

```
                    Uživatel napíše cokoliv kolem krabiček
                                   │
                                   ▼
                  ┌─────────────────────────────────┐
                  │  meal-prep-orchestrator (skill) │
                  │  • zjistí stav projektu         │
                  │  • klasifikuje intent            │
                  │  • vyřeší chybějící předpoklady │
                  │  • spustí správný workflow      │
                  └─────────────────────────────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
       setup-macro-profile  generate-meal-plan  build-shopping-list
                │                  │                  │
                └──────────────────┴──────────────────┘
                                   │
                                   ▼
                    domain skilly (knihovna znalostí)
                  ─────────────────────────────────────
                  rohlik-product-lookup (cache + MCP)
                  waste-minimization (algoritmus)
                  macro-balancing (BMR/TDEE/úpravy)
                  meal-planning (best practices, šablony)
```

## Skilly

### Workflow skilly (auto-aktivace na intent uživatele)

| Skill | Aktivuje se na |
|-------|----------------|
| `meal-prep-orchestrator` | Cokoliv kolem krabiček, makro plánování, Rohlík v kontextu jídel |
| `setup-macro-profile` | „nastav makra", „spočítej kolik mám jíst", „přidej osobu" |
| `generate-meal-plan` | „naplánuj jídelníček", „udělej krabičky na týden", „navrhni recepty" |
| `build-shopping-list` | „udělej nákup", „přidej do košíku", „shopping list" |

### Domain skilly (znalostní moduly, používají se uvnitř workflowů)

| Skill | K čemu |
|-------|--------|
| `rohlik-product-lookup` | Cache-first vyhledávání produktů, atomické zápisy, batch API, TTL pravidla |
| `waste-minimization` | Iterativní algoritmus minimalizace odpadu balených produktů s kompenzací maker |
| `macro-balancing` | BMR/TDEE/cíl výpočty, mikro-úpravy porcí, multi-person scénáře |
| `meal-planning` | Best practices krabičkového vaření, šablony jídel, skladovatelnost |

## Příkazy

Jediný:

| Příkaz | Co dělá |
|--------|---------|
| `/cache-info` | Zobrazí stav lokální cache produktů a receptů, umožní refresh cen, mazání, export do CSV. |

Vše ostatní funguje přes přirozenou konverzaci — orchestrátor pozná intent a spustí správný workflow.

## AGENTS.md

Plugin si v projektu udržuje **managed sekci** v `./AGENTS.md`, do které zapisuje:

- Aktivní profily osob a jejich denní makra
- Globální nastavení (max cena, strategie odpadu)
- Datum + horizont posledního jídelníčku
- Stav cache (počet produktů, nejstarší cena)

Sekce je ohraničená:
```
<!-- meal-prep-rohlik:start -->
... managed obsah ...
<!-- meal-prep-rohlik:end -->
```

Plugin **nikdy nesahá** mimo tyto značky — tvůj vlastní obsah v `AGENTS.md` je v bezpečí.

Díky tomu Claude vždy ví, v jakém stavu projekt je, i v nové konverzaci. Po restartu nebo druhý den prostě napíšeš *„udělej nákup z té poslední"* a plugin ví, který jídelníček máš na mysli.

## MCP server

Plugin používá **oficiální Rohlík MCP server** (`https://mcp.rohlik.cz/mcp`, HTTP). Pokrývá vyhledávání produktů, nutriční tabulky, košík, shopping listy, slevy i recepty.

### Přihlašovací údaje

Plugin potřebuje **přístupové údaje k tvému účtu na Rohlik.cz** v environment variables:

```bash
export ROHLIK_USERNAME="tvuj-email@example.com"
export ROHLIK_PASSWORD="tvoje-heslo"
```

V Cowork tyto proměnné nastavíš v aplikaci → Settings → MCP servery → meal-prep-rohlik.

## Datový adresář

Plugin pracuje s lokální složkou v **pracovní složce uživatele** (ne v plugin složce), aby data zůstávala s projektem a sdílela je celá domácnost:

```
.krabickova-dieta/
├── profiles/
│   ├── filip.json          # makra a profil osoby
│   └── lucka.json
├── settings.json           # max cena/den, pantry, waste strategy
├── cache/
│   ├── products.json       # nutriční hodnoty + ceny z Rohlíku
│   └── recipes.json        # opakovaně použitelné recepty
└── jidelnicky/
    ├── 2026-05-11.md
    └── 2026-05-11-nakup-2026-05-12-08-15.md

AGENTS.md                   # automaticky udržovaná managed sekce
```

## Klíčové vlastnosti

- **Žádné slash commandy si pamatovat.** Stačí napsat, co chceš, jako by tam seděl kuchař.
- **Pouze rohlik.cz** jako zdroj nutričních dat a cen. Hodnoty odpovídají konkrétním produktům, které si koupíš.
- **Cache-first.** Plugin si pamatuje, co už zjistil. Nutrice TTL 30 dní, ceny 24 h.
- **Multi-person.** Více osob s rozdílnými makry — společné vaření, různé porce.
- **Minimalizace odpadu.** Iterativní úprava porcí, aby se spotřebovala celá balení; pokud nelze, odpad směřuje k trvanlivým / levným potravinám podle preference.
- **Maso a ryby = 100 % spotřeba.** Hard constraint — nemrazí se, vždy se musí celé sníst v rámci jídelníčku. Pokud balení neodpovídá potřebě, plugin upraví porce nebo přidá další recept.
- **Cenový strop na den/osobu.** Pokud překročí, plugin nabídne výměnu za levnější alternativy nebo zlevněné položky z Rohlíku.
- **Bezpečné modifikační akce.** Přidání do košíku, shopping listu nebo mazání cache vždy s explicitním potvrzením v chatu.
- **AGENTS.md udržovaný automaticky** — Claude má vždy kontext, i v nové konverzaci.
- **Vše česky.**

## Rychlý start

1. Nainstaluj plugin do Cowork.
2. Nastav environment variables `ROHLIK_USERNAME` a `ROHLIK_PASSWORD`.
3. Napiš *„nastav mi makra"* nebo *„naplánuj jídelníček"* — plugin tě provede.
4. Kdykoliv pak řekni co potřebuješ — plugin pozná z AGENTS.md, v jakém stavu projekt je.

## Poznámka k bezpečnosti

Plugin **nikdy nebude:**
- Zadávat platební údaje.
- Dokončovat objednávku (checkout) — to vždy uděláš ty sám/sama.
- Mazat něco bez explicitního potvrzení.

Plugin **vždy potvrdí** před tím, než:
- Přidá produkty do košíku.
- Vytvoří shopping list.
- Smaže cache nebo profil.

## Changelog

### 0.3.1

- Instrukce pracovní složky jsou v `AGENTS.md`; zdroj pluginu je v `.agents/plugins/meal-prep-rohlik/`.
- Instalační payload a ZIP se sestavují s fyzickými soubory.


### 0.3.0
- **Hard constraint: maso a ryby se nemrazí.** Vždy se musí celé spotřebovat v rámci jídelníčku. `waste-minimization` algoritmus má pro maso/drůbež/ryby zvláštní větev — pokud nelze úpravou porce, přidá další recept nebo vyměňuje balení/recept.
- Odebrán `rohlik-community` MCP server — oficiální `rohlik-official` pokrývá vše potřebné (vyhledávání, nutriční tabulky, košík, shopping listy, slevy, recepty).

### 0.2.0
- Architektura přepracovaná na **orchestrator + auto-aktivované workflow skilly**. Uživatel už si nemusí pamatovat slash commandy.
- Přidán `meal-prep-orchestrator` skill — router intent → workflow.
- Plugin si automaticky udržuje managed sekci v `AGENTS.md` v projektu.
- Příkazy `/makra`, `/jidelnicek`, `/nakup` převedené na auto-aktivované skilly. Zachován jen `/cache-info`.

### 0.1.0
- První verze. Příkazy `/makra`, `/jidelnicek`, `/nakup`, `/cache-info`.
- Skilly `rohlik-product-lookup`, `waste-minimization`, `macro-balancing`, `meal-planning`.
- MCP servery `rohlik-official` (HTTP) a `rohlik-community` (npm).
