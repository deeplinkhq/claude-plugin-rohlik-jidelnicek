---
name: build-shopping-list
description: Vygeneruje nákupní seznam z existujícího jídelníčku s aktuálními cenami z Rohlíku a volitelně přidá do košíku. Aktivuj, když uživatel žádá nákupní seznam, chce přidat věci do košíku Rohlíku, chce shopping list, nebo se odkazuje na hotový jídelníček a říká „nakup", „pošli na Rohlík", „udělej objednávku". Po dokončení aktualizuje AGENTS.md.
---

# Build Shopping List

Pracuje s **jídelníčkem, který už existuje** v `.krabickova-dieta/jidelnicky/`. Cílem je rychle vygenerovat aktualizovaný nákupní seznam (čerstvé ceny) a volitelně přidat do košíku Rohlíku. Veškerá komunikace **česky**.

## Krok 0: Najdi jídelníček

```bash
ls -1t .krabickova-dieta/jidelnicky/*.md 2>/dev/null
```

Pokud žádný jídelníček neexistuje — řekni: *„Nemám žádný jídelníček. Mám rovnou jeden vytvořit?"* a spusť skill `generate-meal-plan` (po něm tento skill ne, generate-meal-plan už nákupní seznam dělá sám).

Pokud existuje jen jeden, použij ho automaticky a oznam to. Pokud je víc, AskUserQuestion s nejnovějšími 4 + možnost „jiný".

Volitelně se zeptej: *„Pro kolik dní z jídelníčku?"* (default = všechny).

## Krok 1: Načti data

- Vybraný markdown jídelníček → vyextrahuj seznam ingrediencí s gramážemi pro každé jídlo, osoby, dny.
- Profily — `.krabickova-dieta/profiles/*.json` (kvůli velikosti porcí, pokud byl jídelníček generovaný pro víc lidí).
- Cache — `.krabickova-dieta/cache/products.json`.
- Settings — `.krabickova-dieta/settings.json` (pantry, max cena, strategie odpadu).

## Krok 2: Sečti potřebu surovin

Pro každou ingredienci sečti totální potřebu (× počet osob × počet dní). Zaokrouhli na rozumné gramy (např. 5 g).

Odečti pantry — pokud uživatel neřekl, že nějakou pantry položku nemá.

## Krok 3: Aktualizuj ceny

Pro každou položku:
1. Cache hit + cena ≤ 24 h stará → použij cache
2. Jinak → `mcp__rohlik-official__get_product_details({productId})` → aktualizuj `price_czk` a `last_price_update` v cache (atomický zápis)

Postupuj podle skillu `@${CLAUDE_PLUGIN_ROOT}/skills/rohlik-product-lookup/SKILL.md`.

## Krok 4: Optimalizace odpadu (pokud potřeba)

Pokud byl jídelníček generován tímto pluginem (`generate-meal-plan`), je už optimalizovaný — jen zaokrouhli nahoru na celá balení.

Pokud byl psán ručně nebo dovezen odjinud, spusť algoritmus z `@${CLAUDE_PLUGIN_ROOT}/skills/waste-minimization/SKILL.md`.

## Krok 5: Cenová kontrola

- Spočítej `total / persons / days` a porovnej s `settings.max_price_per_day_per_person_czk`.
- Pokud překračuje, navrhni výměny dražších položek (`get_discounted_items`).
- Nezvyšuj limit automaticky — informuj uživatele.

## Krok 6: Ukázat seznam

Tabulka v chatu:

| Produkt | Potřeba | Balení | Počet ks | Cena | Zbytek | Link |
|---------|---------|--------|----------|------|--------|------|

Pod tabulkou: **Celková cena** | **Cena/den/osobu** | **Pantry k doplnění**.

Ulož aktualizovaný nákupní seznam jako přílohu k jídelníčku — `.krabickova-dieta/jidelnicky/{soubor-bez-pripony}-nakup-{YYYY-MM-DD-HH-MM}.md`.

Zobraz `computer://...` odkaz.

## Krok 7: Akce — košík nebo shopping list

AskUserQuestion:
- **Co dál?**
  - Přidat do mého košíku na Rohlíku
  - Vytvořit shopping list na Rohlíku
  - Jen mi to ukázat, nic nepřidávej

### Přidat do košíku
**STOP — vyžaduj explicitní potvrzení v chatu** se souhrnem („Přidám N produktů za X Kč"). Po potvrzení `mcp__rohlik-official__add_items_to_cart`.

### Shopping list
`mcp__rohlik-official__create_shopping_list` + `mcp__rohlik-official__add_products_to_shopping_list`.

## Krok 8: Aktualizuj AGENTS.md

Aktualizuj managed sekci — hlavně počet produktů v cache a nejstarší cena (může se změnit po refresh cen). Postup v `@${CLAUDE_PLUGIN_ROOT}/skills/meal-prep-orchestrator/references/agents-md-template.md`.

## Důležité zásady

- **Pantry nezahrnuj** do nákupu (pokud uživatel neřekl jinak).
- **Pouze rohlik.cz** pro ceny.
- **Modifikační akce** (košík, shopping list) **vždy s potvrzením v chatu**.
- **Cache-first** s 24h TTL pro ceny.
- **Komunikuj česky.**
