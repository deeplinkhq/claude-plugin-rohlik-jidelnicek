---
name: rohlik-product-lookup
description: Vyhledává produkty na Rohlik.cz a kešuje jejich nutriční hodnoty, ceny a velikosti balení. Použij vždy, když je potřeba získat data o potravině z Rohlíku — při sestavování jídelníčku, kontrole cen, generování nákupního seznamu nebo aktualizaci cache.
---

# Rohlik Product Lookup

Tento skill vysvětluje **jak používat Rohlik MCP** efektivně a jak fungovat s lokální cache produktů.

## Cache-first princip

**Vždy nejprve cache, pak MCP.** MCP volání jsou drahá (latence + tokeny + rate limit). Cache je v souboru `.krabickova-dieta/cache/products.json` v pracovní složce uživatele.

### Schéma cache

```json
{
  "1234567": {
    "productId": "1234567",
    "name": "Kuřecí prsní řízky bez kosti",
    "brand": "Vodňanské kuře",
    "category": "maso",
    "nutrition_per_100g": {
      "energy_kcal": 110,
      "protein_g": 23.1,
      "carbs_g": 0.0,
      "sugars_g": 0.0,
      "fat_g": 1.3,
      "saturated_fat_g": 0.4,
      "fiber_g": 0.0,
      "salt_g": 0.1
    },
    "package_size_g": 500,
    "package_unit": "balení",
    "price_czk": 149.90,
    "price_per_kg_czk": 299.80,
    "last_price_update": "2026-05-11",
    "nutrition_cached_at": "2026-04-01",
    "url": "https://www.rohlik.cz/1234567-vodnanske-kure-kureci-prsni-rizky"
  }
}
```

### TTL pravidla

- **Nutriční hodnoty:** 30 dní (mění se zřídka, jen při reformulaci produktu).
- **Ceny:** 24 hodin (Rohlík je mění běžně, akce, sezóna).
- **Package size:** stabilní, neaktualizovat dokud produkt existuje.

## Použití MCP nástrojů

Plugin používá **oficiální Rohlík MCP server** (`rohlik-official`, HTTP).

| Tool | Použití |
|------|---------|
| `batch_search_products({queries: [...]})` | Najít produkty podle názvů. Vrací seznamy hits s `productId`, `name`, `price`, `pricePerUnit`. |
| `get_product_details({productId})` | Detail jednoho produktu — cena, popis, balení. |
| `get_products_composition_batch({productIds: [...]})` | **Klíčové** — nutriční tabulka per 100 g pro až N produktů najednou. |
| `get_discounted_items()` | Aktuální slevy — užitečné pro „levnější alternativu". |
| `get_all_user_favorites()` | Oblíbené uživatele — preferuj při generování receptů. |
| `search_recipes_by_vector_similarity({query})` | Vyhledávání receptů z databáze Rohlíku (vector similarity). |
| `get_recipe_detail({recipeId})` | Detail receptu z Rohlíku. |
| `add_items_to_cart({items: [{productId, quantity}]})` | Přidat do košíku (modifikační, vyžaduje potvrzení). |
| `create_shopping_list({name})` + `add_products_to_shopping_list` | Shopping list (modifikační, vyžaduje potvrzení). |

## Workflow vyhledání produktu

```
ingredience "kuřecí prsa 400 g"
    │
    ▼
najdi v cache podle "kuřecí prs" v name (case-insensitive)
    │
    ├── HIT (nutrition < 30 d, price < 24 h)  →  použij cache
    │
    ├── HIT (price > 24 h)                    →  get_product_details, refresh cena
    │
    └── MISS                                  →  batch_search_products
                                                  ├── filtruj kvalitní hit
                                                  │   (preferuj domácí brand,
                                                  │    BIO/farmářský dle profilu)
                                                  ├── get_products_composition_batch
                                                  └── ulož do cache
```

## Best practices

- **Batchuj požadavky.** Pokud potřebuješ data pro 10 produktů, zavolej `batch_search_products` jednou s 10 query, ne 10×.
- **Filtruj inteligentně.** Hledáš-li „mléko", `batch_search_products({queries: ["mléko polotučné 1l"]})` dá lepší výsledek než `["mléko"]`.
- **Pamatuj alternativy.** Pokud uživatel preferuje BIO, hledej s tagem („BIO mléko") a fall-back na klasiku.
- **Cache zápis atomicky:** napiš do `.krabickova-dieta/cache/products.tmp.json`, pak `mv` na `products.json`. Předejde to korupci při crashi.

```bash
# atomický zápis cache (příklad)
TMP=".krabickova-dieta/cache/products.tmp.json"
FINAL=".krabickova-dieta/cache/products.json"
echo "$JSON" > "$TMP" && mv "$TMP" "$FINAL"
```

- **Persistuj často.** Po každých ~10 nových produktech zapiš cache, aby se práce neztratila.
- **Loguj cache hit rate.** Pokud je < 50 %, něco je špatně (např. uživatel maže cache, nebo se mění klíče).

## Pouze rohlik.cz

**Nikdy nepoužívej WebFetch ani WebSearch pro získání nutričních dat.** Důvod: konzistence (jiné zdroje mají jiné hodnoty pro stejnou potravinu), aktuálnost, a uživatel kupuje na Rohlíku — chce hodnoty toho konkrétního produktu, ne obecné.

Pokud produkt na Rohlíku není (zkus 2 alternativní queries než to vzdáš), nabídni uživateli:
1. **Náhradu** ze stejné kategorie, která na Rohlíku je
2. **Označení odhadem** v jídelníčku, aby věděl, že hodnoty nejsou ověřené
