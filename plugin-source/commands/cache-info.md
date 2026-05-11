---
description: Zobrazit a spravovat lokální cache produktů a receptů
allowed-tools: Read, Write, Edit, Bash, AskUserQuestion
---

# Správa cache

Tento příkaz dává přehled o lokální cache, kterou plugin používá, a umožňuje její údržbu (vyčistit, refreshnout, exportovat).

## Krok 0: Načti cache

```bash
ls -la .krabickova-dieta/cache/ 2>/dev/null
```

Načti:
- `.krabickova-dieta/cache/products.json` (klíč: `productId`, hodnota: `{ name, brand, nutrition_per_100g, package_size_g, price_czk, last_price_update, cached_at }`)
- `.krabickova-dieta/cache/recipes.json`

## Krok 1: Zobraz souhrn

Stručná tabulka:
- Počet produktů v cache + nejstarší/nejnovější `cached_at`
- Počet produktů s **cenou starší než 1 den** (kandidát na refresh)
- Počet produktů s **nutricí starší než 30 dní**
- Počet uložených receptů, top 5 nejpoužívanějších (podle `use_count`)
- Velikost cache souborů na disku

## Krok 2: Akce

Pomocí AskUserQuestion (multiSelect: true):
- **Vypsat detailně všechny produkty / recepty**
- **Refreshnout staré ceny** (volá MCP pro produkty s cenou > 24 h)
- **Smazat konkrétní položku** (zeptej se na productId nebo název)
- **Smazat celou cache produktů** (vyžaduje potvrzení)
- **Smazat celou cache receptů** (vyžaduje potvrzení)
- **Exportovat cache do CSV** (pro vlastní analýzu)
- **Jen zavřít**

### Refresh cen

Pro každý produkt s `last_price_update` starším než 24 h:
1. `mcp__rohlik-official__get_product_details(productId)`
2. Aktualizuj `price_czk` a `last_price_update`
3. Zachovej nutriční data (ta se nemění)

Po dokončení vypiš, kolik položek bylo aktualizováno.

### Mazání

Vždy vyžaduj **explicitní potvrzení v chatu** (vypiš, co se chystá smazat) než smažeš. Smazané položky nelze obnovit (jen znovu nakešovat z MCP).

### Export CSV

Vytvoř `.krabickova-dieta/cache/products-export-{YYYY-MM-DD}.csv` se sloupci:
`productId, name, brand, kcal_per_100g, protein_g, carbs_g, fat_g, package_size_g, price_czk, last_price_update`

Zobraz odkaz `computer://...` na soubor.

## Důležité zásady

- **Cache je v pracovní složce uživatele**, ne v plugin složce.
- **Komunikuj česky.**
- **Mazání vyžaduje potvrzení.**
- **Ceny se mění — refreshuj** před každým novým nákupem.
- **Nutriční hodnoty jsou stabilní** — refreshuj jen jednou za 30 dní.
