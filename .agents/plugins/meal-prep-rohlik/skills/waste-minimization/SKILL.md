---
name: waste-minimization
description: Algoritmus pro minimalizaci odpadu při sestavování jídelníčku z balených surovin. Použij vždy při sestavování nákupního seznamu, když je potřeba sladit gramáže receptů s velikostí balení produktů, a chytře využít přebytek bez rozhození maker.
---

# Minimalizace odpadu

Cílem je, aby se **spotřebovala celá balení** a vyhodil se minimální zbytek. Pokud minimum nejde, **odpad směřuj k trvanlivým / levným potravinám** podle uživatelovy strategie.

## Vstup

- Recepty se surovinami (gramáže × osoby × dny) → **surová potřeba** v gramech.
- Cache produktů s `package_size_g`.
- `settings.waste_strategy`: `"trvanlivé potraviny"` | `"levné potraviny"` | `"absolutní minimum"`.
- `settings.max_price_per_day_per_person_czk`.
- Cílová makra každé osoby (tolerance ±10 %).

## Hard constraints (přednost před vším ostatním)

### 🥩 Maso a ryby — vždy zpracovat celé balení v jídelníčku

**Maso (kuřecí, krůtí, hovězí, vepřové, jehněčí, mleté), drůbež a ryby/mořské plody se NIKDY nemrazí jako fallback.** Vždy musí být **celé spotřebovány v rámci plánovaného jídelníčku**.

To znamená:

1. Pokud `need_g[maso] < buy_g[maso]` (balení je větší než potřeba), **musíš** to vyřešit v iteraci — nikdy neakceptuj zbytek masa.
2. Pokud iterace nedokáže navýšit porce v existujících receptech tak, aby se balení dorovnalo do tolerance maker, **přidej do jídelníčku další recept**, který zbylé maso pojme.
3. Pokud ani přidaný recept nestačí, **vyměň balení** za menší (např. 250 g místo 500 g), nebo **vyměň recept** za jiný, který se přesněji trefí.
4. **Nikdy nesmí v reportu skončit jako "do mrazáku"** ani jako "pantry surplus" — to platí jen pro trvanlivé potraviny.

Důvod: maso a ryby se v běžné rodinné krabičkové dietě hromadí v mrazáku, kde se zapomenou. Tahle constraint nutí jídelníček navrhnout reálně použitelně.

### Postup pro maso (závazný)

```
need_g[maso] = X
buy_g[maso]  = ceil(X / package_size) × package_size

if buy_g > need_g:
    diff = buy_g - need_g
    # Pokus 1: navýšit porce v existujících receptech s tímto masem
    if can_distribute(diff, existing_recipes_with_meat, macro_tolerance=10%):
        apply_distribution()
    # Pokus 2: přidat nový recept využívající přesně diff gramů
    elif new_recipe_fits(diff, day_macros, time_limit):
        add_recipe(diff)
    # Pokus 3: zkusit menší balení (jiný produkt na Rohlíku)
    elif smaller_package_exists(meat):
        switch_package()
    # Pokus 4: vyměnit recept za takový, jehož potřeba lépe sedí
    else:
        swap_recipe(meat_amount=multiple_of_package_size)

# NIKDY: accept_waste(meat) nebo freeze(meat)
```

## Výstup

- Finální nákupní seznam (počet balení).
- Upravené gramáže v receptech.
- Přepočtená makra (musí stále sedět ±10 %).
- Hlášení: kolik gramů zbývá nevyužito a kde.

## Algoritmus (iterativní)

### 1. Surový výpočet potřeby

```
need_g[product] = sum(amount_g × persons × days) for product across all meals
```

### 2. Zaokrouhlení na celá balení

```
packs[product] = ceil(need_g[product] / package_size_g[product])
buy_g[product]  = packs[product] × package_size_g[product]
waste_g[product] = buy_g[product] - need_g[product]
waste_pct[product] = waste_g / buy_g
```

### 3. Klasifikuj zbytky

| Kategorie | Práh | Akce |
|-----------|------|------|
| Akceptovatelné | waste_pct ≤ 15 % | Necháme být. |
| Optimalizovat | 15 % < waste_pct ≤ 50 % | Iteruj — pokus se navýšit potřebu. |
| Velký odpad | waste_pct > 50 % | Akce dle waste_strategy (viz níže). |

### 4. Iterace (max 5×)

Pro každý produkt v kategorii „Optimalizovat":

1. **Vyber recept**, který tento produkt používá.
2. **Spočítej**, o kolik g navýšit množství v receptu, aby `need_g = buy_g` (plné balení).
3. **Navýšení převeď na kalorie** přes nutrition_per_100g.
4. **Kompenzuj jinde:** odečti odpovídající kcal z jiných ingrediencí v tomtéž jídle (nebo jiných jídel téhož dne) — preferuj přílohy (rýže, brambory, těstoviny).
5. **Zkontroluj makra po úpravě.** Pokud odchylka > 10 %, vrať změnu zpět a zkus jiný recept.
6. **Persistuj nové gramáže** v jídelníčku.

Pokud po 5 iteracích nelze odpad snížit pod 15 %, jdi na krok 5.

### 5. Akce pro velký odpad

#### Strategie `"trvanlivé potraviny"` (default)

Pokud zbude >50 % balení trvanlivé potraviny (rýže, těstoviny, mouka, luštěniny, koření, oleje, ořechy, müsli) — **necháme**, uživatel ji použije příště. Označ v hlášení jako „pantry surplus".

Pokud zbude >50 % balení **netrvanlivé rostlinné** potraviny (mléčné, čerstvá zelenina/ovoce, pečivo) → najdi recept navíc, který suroviny dojí.

**Maso, drůbež a ryby — viz Hard constraints výše.** Mrazení není přípustné, vždy se musí spotřebovat v rámci jídelníčku.

#### Strategie `"levné potraviny"`

Akceptuj odpad pouze u položek pod cenovým prahem (např. < 30 Kč/balení). Drahé položky (maso > 100 Kč) iteruj do konce.

#### Strategie `"absolutní minimum"`

Pokud nelze dorovnat balení úpravou porcí, **úplně vyměň recept** za jiný, který suroviny pojme. Pomalejší, ale nejmenší odpad.

### 6. Cenová kontrola

Po optimalizaci spočítej `total_price / persons / days`. Pokud > limit:
1. Najdi nejdražší položku v košíku.
2. `get_discounted_items` z Rohlíku — je v akci alternativa?
3. Pokud ano, vyměň a přepočítej makra.
4. Pokud ne, navrhni levnější ingredienci ze stejné kategorie (např. tofu místo lososa).
5. Pokud ani tak — informuj uživatele s návrhem (nezvyšuj limit automaticky).

## Praktické příklady

### Příklad 1 — kuřecí prsa 500 g balení

- Recept: 4× oběd × 1 osoba × 150 g = 600 g potřeba.
- Balení 500 g → koupit 2 ks = 1000 g, odpad 400 g (40 %).
- Iterace: navýšit porci kuřete z 150 → 250 g/oběd. Nová potřeba = 1000 g, odpad 0 %.
- Kompenzace: 250 g kuřete = +110 kcal vs. 150 g. Odečti 30 g rýže × 4 obědy.
- Kontrola maker: bílkoviny mírně vyšší (OK pro hubnoucí cíl), kalorie sedí.

### Příklad 2 — feta 150 g balení, recept 50 g do salátu

- Potřeba 50 g, koupit 150 g, odpad 100 g (67 %).
- Strategie „trvanlivé": feta není trvanlivá → najdi další využití.
- Návrh: přidej salát s fetou další den, nebo zapečené brambory s fetou.
- Pokud uživatel ne — strategie „levné": feta je středně drahá, akceptujeme jen pokud pod 60 Kč.

### Příklad 3 — rýže basmati 1 kg balení, potřeba 320 g

- Odpad 680 g (68 %), ale **rýže je trvanlivá** → strategie „trvanlivé potraviny" → akceptuj, nemění se.
- Označ ve výstupu: „⚠ Rýže basmati 1 kg — využiješ na 320 g, zbytek (680 g) zůstane do zásoby."

## Reporting

Na konci optimalizace v jídelníčku zobraz:

```
Optimalizace odpadu — výsledek po 3 iteracích:

✓ Kuřecí prsa 500 g — porce navýšena ze 150 g na 250 g (kompenzováno -30 g rýže)
✓ Mleté hovězí 400 g — přidán recept "Boloňská omáčka na pondělí" pro využití zbytku
✓ Tvaroh 250 g — porce navýšena z 200 g na 250 g (kompenzováno -10 g müsli)
○ Rýže basmati 1 kg — pantry surplus 680 g (akceptováno, trvanlivá)
○ Feta 150 g — přidán recept "Pečené brambory s fetou" pro využití
⚠ Špenát čerstvý 200 g — zbytek 70 g zařazen do středeční svačiny (smoothie)

Celkový odpad: 5 % nákupu (před: 27 %)
Maso a ryby: 100 % spotřebováno v jídelníčku ✓
```

V reportu je **kontrolní řádek** „Maso a ryby: 100 % spotřebováno" — pokud nesedí, optimalizace není hotová a musí se vrátit na začátek.

## Důležité zásady

- **Maso a ryby = 100 % spotřeba v jídelníčku.** Nikdy mrazák, nikdy "pantry surplus". Pokud nelze úpravou porce, přidej recept; pokud ne, vyměň balení nebo recept. Hard constraint, viz sekce výše.
- **Makra mají přednost před zbytky rostlinných surovin.** Optimalizace odpadu (mimo maso) nesmí rozhodit makra > 10 %. Pokud rozhodí → vrátit změnu.
- **Trvanlivé potraviny ≠ odpad.** Zbylá rýže/těstoviny/koření se v reportu nepočítají jako waste.
- **Maximálně 5 iterací pro rostlinné suroviny**, jinak akcept a posun dál — uživatel chce výsledek, ne nekonečnou optimalizaci. Pro maso iteruj **dokud nesedí** (nebo dej up explicitní hlášku „nelze sestavit, vyměň balení/recept").
- **Loguj každou iteraci** — uživatel chce vidět, co a proč se měnilo.
