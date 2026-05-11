---
name: generate-meal-plan
description: Sestaví krabičkový jídelníček s makry a nákupním seznamem z Rohlíku. Aktivuj, když uživatel chce naplánovat jídelníček, vytvořit krabičky na týden, navrhnout meal-prep recepty, sestavit menu s ohledem na makra, nebo když popisuje, na co má chuť a chce z toho jídla na několik dní. Po dokončení aktualizuje CLAUDE.md. Hard constraint: maso a ryby se nemrazí — vždy musí být celé spotřebovány v jídelníčku.
---

# Generate Meal Plan

Hlavní workflow plánování. Vždy končí markdown souborem v `.krabickova-dieta/jidelnicky/{datum}.md` a aktualizovanou managed sekcí v `CLAUDE.md`.

Pro detaily algoritmů použij další skilly:
- `@${CLAUDE_PLUGIN_ROOT}/skills/rohlik-product-lookup/SKILL.md` — vyhledávání produktů a cache
- `@${CLAUDE_PLUGIN_ROOT}/skills/waste-minimization/SKILL.md` — minimalizace odpadu
- `@${CLAUDE_PLUGIN_ROOT}/skills/macro-balancing/SKILL.md` — vážení maker
- `@${CLAUDE_PLUGIN_ROOT}/skills/meal-planning/SKILL.md` — best practices krabičkování

Veškerá komunikace **česky**.

## Krok 0: Načti vše potřebné

```bash
mkdir -p .krabickova-dieta/profiles .krabickova-dieta/cache .krabickova-dieta/jidelnicky
```

1. Profily — `.krabickova-dieta/profiles/*.json`. **Pokud žádný neexistuje**, řekni *„Nejdřív potřebuju aspoň jeden profil."* a spusť skill `setup-macro-profile`. Po jeho dokončení se vrať sem.
2. Settings — `.krabickova-dieta/settings.json` (max cena/den, pantry, strategie odpadu).
3. Cache produktů — `.krabickova-dieta/cache/products.json` (pokud chybí, vytvoř `{}`).
4. Cache receptů — `.krabickova-dieta/cache/recipes.json` (pokud chybí, vytvoř `{}`).

## Krok 1: Zjisti rozsah jídelníčku

AskUserQuestion (max 4 otázky najednou):

**Kolo 1:**
- **Pro které osoby vařit?** zobraz seznam profilů (multiSelect: true)
- **Které chody zahrnout?** snídaně / svačina / oběd / večeře (multiSelect: true). Předvyplň průnik z profilů.
- **Na kolik dní?** 1 / 3 / 5 / 7 / jiné (volný text)
- **Mají všechny osoby jíst stejné jídlo?** ano (společné vaření, různé porce) / ne (každý své)

**Kolo 2 — chuť a omezení:**
- **Na co máte tento týden chuť?** (volný text — typ kuchyně, hlavní suroviny, styl)
- **Něco extra, čemu se vyhnout tento týden?** (volný text, nad rámec profilu)
- **Vařit ze zbytků z minulého týdne?** ano (vypiš co máš) / ne / přeskočit
- **Časový limit přípravy:** rychle (do 20 min) / standard (30–45 min) / čas mám (45+)

## Krok 2: Pantry check

Z `settings.json` načti pantry list. Zeptej se:
- *„Máte všechny tyto základní suroviny doma?"*
  - Ano vše mám
  - Chybí mi (volný text)

Chybějící pantry položky se přidají do nákupu.

## Krok 3: Návrh jídelníčku (draft)

Na základě maker, preferencí, počtu jídel:

Pro každý den a chod:
- **Název receptu**
- **Stručný popis** (1–2 věty)
- **Hlavní ingredience** (zatím odhadem)
- **Odhadované makra na porci** (kcal | B | S | T)
- **Doba přípravy**

**Přednost recepty z cache** `.krabickova-dieta/cache/recipes.json` (úspora + osvědčené chuťovky). Pokud cache obsahuje vhodný recept, zařaď ho a označ „📂 z cache".

**Kontrola maker:** sečti denní makra a porovnej s cíli každé osoby (tolerance ±10 % pro den, ±20 % pro jednotlivý chod). Pokud nesedí, přepiš návrh.

Prezentuj draft v tabulce + zeptej se:
- *„Jak se ti to líbí?"*
  - Pokračovat s tímto návrhem
  - Vyměnit konkrétní jídlo (řekni které a za co)
  - Vygenerovat nový návrh s jinou náladou

## Krok 4: Reálná data z Rohlíku

Pro každou ingredienci postupuj podle skillu `rohlik-product-lookup`. Plugin používá **oficiální Rohlík MCP** (`mcp__rohlik-official__*`):

1. Cache hit (nutrition < 30 d, cena < 1 d) → použij cache
2. Cache hit s starou cenou → `mcp__rohlik-official__get_product_details`, refresh ceny
3. Cache miss → `mcp__rohlik-official__batch_search_products` + `mcp__rohlik-official__get_products_composition_batch`, ulož do cache

**Persistuj cache** atomicky (`.tmp` + `mv`) po každých ~10 produktech.

**Pouze rohlik.cz** jako zdroj dat. Pokud produkt na Rohlíku není (zkus 2 alternativní queries), nabídni alternativu nebo označ makra jako odhad.

## Krok 5: Přesné makra

S reálnými daty přepočítej:
- Pro každou ingredienci: `(množství_g / 100) × hodnota_per_100g`
- Sečti za recept → den → osobu
- Tabulka **cíl vs reálnost** + **% odchylka** pro každou osobu
- Pokud > 10 %, navrhni mikro-úpravy (`macro-balancing` má playbook)

## Krok 6: Optimalizace odpadu

Striktně podle algoritmu v `waste-minimization`:

1. Surová potřeba × všechny dny × osoby
2. Zaokrouhli nahoru na celá balení
3. **Maso, drůbež a ryby — Hard constraint:** vždy 100 % spotřeba. Pokud balení > potřeba, **musíš** navýšit porce nebo přidat recept (max iterací neplatí, mrazák zakázán). Pokud nejde — vyměň balení nebo recept.
4. **Rostlinné suroviny:** iteruj max 5×, cíl < 15 % odpadu / nebo akceptace „pantry surplus" pro trvanlivé.
5. Loguj každou iteraci stručně (co, proč).
6. V reportu zobraz kontrolní řádek „Maso a ryby: 100 % spotřebováno ✓" — pokud nesedí, není hotovo.

## Krok 7: Cenová kontrola

- Spočítej cenu/den/osobu.
- Pokud > `settings.max_price_per_day_per_person_czk`:
  - `mcp__rohlik-official__get_discounted_items` — zlevněné alternativy
  - Vyměň drahé položky, přepočítej makra
  - Pokud nelze — informuj uživatele s návrhem (nezvyšuj limit automaticky)

## Krok 8: Ulož finální markdown

`.krabickova-dieta/jidelnicky/{YYYY-MM-DD}.md`:

```markdown
# Krabičkový jídelníček YYYY-MM-DD ({N} dní, {osoby})

## Souhrn
- Osoby: ...
- Chody: ...
- Cena celkem: XXX Kč | Cena/den/osobu: XXX Kč
- Celkový odpad: X % nákupu

## Den 1 — pondělí

### Snídaně — {název}
**Pro {osoba}:** {ingredience s gramáží}
**Postup:** ...
**Makra:** kcal | B | S | T

### Oběd — {název}
...

**Denní součet pro {osoba}:** kcal | B | S | T
**Cíl:** kcal | B | S | T
**Odchylka:** ±X %

## Nákupní seznam (Rohlik.cz)

| Produkt | Potřeba | Balení | Počet ks | Cena | Zbytek | Link |
|---------|---------|--------|----------|------|--------|------|
| ... | ... | ... | ... | ... | ... | [↗](https://www.rohlik.cz/...) |

**Celková cena:** XXX Kč
**Pantry položky k doplnění:** ...
```

Zobraz uživateli `computer://...` odkaz na soubor.

## Krok 9: Aktualizuj CLAUDE.md

**Vždy** aktualizuj managed sekci v `./CLAUDE.md`:
- Datum + horizont nového jídelníčku
- Aktualizovaný stav cache (počet produktů + nejstarší cena)

Postup je v `@${CLAUDE_PLUGIN_ROOT}/skills/meal-prep-orchestrator/references/claude-md-template.md`.

## Krok 10: Akce — košík nebo shopping list

AskUserQuestion:
- **Co dál?**
  - Přidat produkty do mého košíku na Rohlíku
  - Vytvořit shopping list na Rohlíku
  - Stačí mi jen ten dokument

### Přidat do košíku
**STOP — vyžaduj explicitní potvrzení v chatu** se souhrnem („Přidám N produktů za X Kč"). Po potvrzení `mcp__rohlik-official__add_items_to_cart`. Zobraz odkaz na košík https://www.rohlik.cz/kosik.

### Shopping list
`mcp__rohlik-official__create_shopping_list({name: "Krabičky {datum}"})` + `mcp__rohlik-official__add_products_to_shopping_list`.

## Krok 11: Cache receptů

Každý úspěšně použitý recept ulož do `.krabickova-dieta/cache/recipes.json`:

```json
{
  "kureci-stripsy-s-ryzi": {
    "name": "Kuřecí stripsy s rýží a brokolicí",
    "tags": ["rychle", "kuře", "fitness"],
    "time_min": 25,
    "servings": 1,
    "ingredients": [
      { "product_id": "1234567", "name": "Kuřecí prsa", "amount_g": 150 },
      { "product_id": "7654321", "name": "Rýže basmati", "amount_g": 80 }
    ],
    "macros_per_serving": { "kcal": 540, "protein_g": 45, "carbs_g": 60, "fat_g": 9 },
    "instructions": "...",
    "last_used": "2026-05-11",
    "use_count": 3
  }
}
```

## Důležité zásady

- **Vše do `./.krabickova-dieta/`** v pracovní složce.
- **Pouze rohlik.cz** pro nutrice a ceny.
- **Cache-first.**
- **Atomické zápisy.**
- **Modifikační akce** (košík, shopping list) **vždy s potvrzením v chatu**.
- **CLAUDE.md aktualizuj vždy** po vygenerování jídelníčku.
- **Komunikuj česky.**
