---
name: meal-prep-orchestrator
description: Hlavní router pro krabičkovou dietu. Aktivuj se vždy, když uživatel mluví o krabičkách, jídelníčku, meal-prep plánování, makrech, nákupu potravin na týden, vyvažování živin nebo o Rohlíku v souvislosti s plánováním jídel. Zjisti aktuální stav (profily, jídelníčky, cache) a spusť odpovídající workflow — setup-macro-profile, generate-meal-plan nebo build-shopping-list. Komunikuj česky.
---

# Meal Prep Orchestrator

Tento skill je **vstupní bod** pro celý workflow krabičkové diety. Uživatel ti řekne přirozenou řečí, co chce ("naplánuj mi jídelníček na týden", "kolik mám jíst bílkovin", "udělej nákup z toho jídelníčku"...) a tvůj úkol je:

1. **Zjistit stav** — co už v projektu existuje
2. **Rozpoznat intent** — co uživatel chce
3. **Vyřešit chybějící předpoklady** — pokud něco chybí, doplnit
4. **Spustit správný workflow skill**

Veškerá komunikace probíhá **česky**.

## Krok 0: Sken stavu

```bash
mkdir -p .krabickova-dieta/profiles .krabickova-dieta/cache .krabickova-dieta/jidelnicky
```

Načti rychle (bez výpisu uživateli):

| Co | Kde | Co s tím |
|----|-----|----------|
| Profily osob | `.krabickova-dieta/profiles/*.json` | seznam jmen + počet |
| Settings | `.krabickova-dieta/settings.json` | existuje? max cena/den? |
| Jídelníčky | `.krabickova-dieta/jidelnicky/*.md` | nejnovější datum + horizont |
| Cache produktů | `.krabickova-dieta/cache/products.json` | počet, nejstarší cena |
| Cache receptů | `.krabickova-dieta/cache/recipes.json` | počet receptů |
| CLAUDE.md sekce | `./CLAUDE.md` | obsahuje `<!-- meal-prep-rohlik:start -->`? |

Pokud `CLAUDE.md` ještě neexistuje, neaktualizuj ho zatím — to udělá `setup-macro-profile` při prvním profilu.

## Krok 1: Klasifikuj intent

Z poslední uživatelovy zprávy rozhodni, do které kategorie patří:

| Intent | Klíčová slova / signály | Akce |
|--------|--------------------------|------|
| **Setup makra** | "nastav makra", "kolik bych měl jíst", "spočítej mi BMR", "vytvoř profil", "přidej osobu", "změnit váhu/cíl", profil neexistuje | → spusť `setup-macro-profile` |
| **Plánování jídelníčku** | "jídelníček", "naplánuj jídla", "krabičky na týden", "co budu jíst", "meal-prep", "navrhni recepty" | → spusť `generate-meal-plan` |
| **Nákupní seznam** | "nákupní seznam", "co mám koupit", "přidej do košíku", "objednat z Rohlíku", explicitně se odkazuje na existující jídelníček | → spusť `build-shopping-list` |
| **Cache management** | "smaž cache", "refreshni ceny", "co je v cache" | → odkaž na `/cache-info` |
| **Otázka na výpočet** | "kolik kalorií má X", "kolik B/S/T denně potřebuju" | → použij skill `macro-balancing` přímo, nespouštěj plný workflow |
| **Otázka na produkt** | "kolik stojí X na Rohlíku", "jaké jsou nutriční hodnoty Y" | → použij skill `rohlik-product-lookup` přímo |
| **Nejasný intent** | nejde určit | → polož **jednu** krátkou ujasňovací otázku |

## Krok 2: Vyřeš chybějící předpoklady

Než spustíš workflow, ověř závislosti:

### `generate-meal-plan` vyžaduje:
- ≥ 1 profil v `.krabickova-dieta/profiles/`
- Pokud chybí: stručně řekni *„Nejdřív potřebuju aspoň jeden makro profil. Pojďme to udělat."* → spusť `setup-macro-profile`, po dokončení automaticky pokračuj v plánování.

### `build-shopping-list` vyžaduje:
- ≥ 1 soubor v `.krabickova-dieta/jidelnicky/`
- Pokud chybí: navrhni *„Žádný jídelníček zatím nemáme. Mám rovnou jeden vytvořit?"* → spusť `generate-meal-plan` (a po něm automaticky `build-shopping-list` pro ten nový).

### `setup-macro-profile` nemá závislosti.

## Krok 3: Krátké hlášení stavu před akcí

Když pokračuješ do workflow, řekni uživateli **jednu větu** o stavu, ať ví, co máš k dispozici. Příklad:

> Mám tvůj profil (Filip, hubnout, 2376 kcal/den) a 47 produktů v cache. Pojďme naplánovat jídelníček.

Pak hned spusť odpovídající skill — uživatel by měl mít pocit, že se „prostě stalo, co chtěl".

## Krok 4: Po dokončení workflow

Vždy aktualizuj **managed sekci** v `./CLAUDE.md` v projektu (postupem z `references/claude-md-template.md`). Sekce obsahuje:
- Seznam profilů a jejich klíčové parametry
- Datum a rozsah posledního jídelníčku
- Stav cache (počet, nejstarší cena)

Sekce je vždy ohraničená:
```
<!-- meal-prep-rohlik:start -->
... managed obsah ...
<!-- meal-prep-rohlik:end -->
```

**Nikdy nesahej** na obsah mimo tyto značky.

## Krok 5: Ujasňující otázka jen když je nutná

Pokud intent je opravdu nejasný, polož **jednu** otázku přes AskUserQuestion s nejvýše 4 možnostmi:

> Co bys ode mě chtěl/a?
> - Naplánovat jídelníček
> - Nastavit / upravit makra
> - Vygenerovat jen nákupní seznam
> - Něco jiného

Nepokládej víc otázek najednou — to ať udělá až workflow skill, který se spustí.

## Příklady auto-routingu

| Uživatel řekne | Routing |
|----------------|---------|
| „Naplánuj mi jídelníček na 5 dní" | profil je → `generate-meal-plan` |
| „Naplánuj mi jídelníček" + není profil | → `setup-macro-profile` → po dokončení `generate-meal-plan` |
| „Kolik bych měl jíst kalorií?" | profil je → odpověz z profilu, nepokládej otázky |
| „Kolik bych měl jíst kalorií?" + není profil | → `setup-macro-profile` |
| „Přidej to do košíku Rohlíku" + je jídelníček | → `build-shopping-list` |
| „Kolik stojí kuřecí prsa?" | → `rohlik-product-lookup` (samostatně), neutekej do plánování |
| „Smaž cache produktů" | → odkaž na `/cache-info` (vyžaduje potvrzení) |

## Důležité zásady

- **Nesnaž se uhádnout** — pokud intent není jasný, raději polož jednu otázku.
- **Nikdy nezahajuj plný workflow bez splněných předpokladů** — vyřeš dependencies.
- **Vždy aktualizuj CLAUDE.md** po dokončení workflow.
- **Komunikuj česky.**
- **Buď stručný** — orchestrátor by měl být neviditelný; uživatel chce výsledek, ne report o směrování.
