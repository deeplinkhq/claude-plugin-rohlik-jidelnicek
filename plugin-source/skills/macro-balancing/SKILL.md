---
name: macro-balancing
description: Vzorce a logika pro výpočet a vyvažování makronutrientů (kcal, bílkoviny, sacharidy, tuky) pro krabičkovou dietu. Použij při výpočtu cílových maker osoby (BMR/TDEE/cíl), při kontrole, zda jídelníček sedí na cíle, a při návrhu mikro-úprav porcí.
---

# Macro Balancing

## BMR — Mifflin-St Jeor rovnice

**Muži:**
```
BMR = (10 × hmotnost_kg) + (6.25 × výška_cm) − (5 × věk) + 5
```

**Ženy:**
```
BMR = (10 × hmotnost_kg) + (6.25 × výška_cm) − (5 × věk) − 161
```

## TDEE — celkový denní výdej

```
TDEE = BMR × faktor_aktivity
```

| Úroveň aktivity | Faktor |
|-----------------|--------|
| Sedavá (kancelář, minimum pohybu) | 1.2 |
| Mírně aktivní (cvičení 1–3× týdně) | 1.375 |
| Aktivní (cvičení 3–5× týdně) | 1.55 |
| Velmi aktivní (cvičení 6–7× týdně) | 1.725 |
| Extrémně aktivní (fyzická práce + cvičení) | 1.9 |

## Cílový kalorický příjem

| Cíl | Úprava TDEE |
|-----|-------------|
| Hubnout (mírně, udržitelně) | TDEE − 400 až 500 kcal |
| Hubnout (agresivně) | TDEE − 600 až 750 kcal (max) |
| Udržovat váhu | TDEE |
| Nabírat lean | TDEE + 200 až 300 kcal |
| Nabírat (bulk) | TDEE + 400 až 500 kcal |

**Bezpečnostní limity:**
- Pod 1200 kcal/den (ženy) nebo 1500 kcal/den (muži) **nedoporučovat**.
- Při BMI < 18.5 nedoporučovat hubnutí.
- Maximální zdravá ztráta váhy ~ 0.7–1 kg/týden.

## Rozložení makronutrientů

### Bílkoviny

| Cíl | g/kg tělesné váhy |
|-----|--------------------|
| Hubnout (důraz na svaly) | 2.0–2.4 |
| Udržovat | 1.6–2.0 |
| Nabírat | 1.8–2.2 |

`1 g bílkoviny = 4 kcal`

### Tuky

- **Minimum:** 0.8 g/kg (hormonální zdraví, vstřebávání vitaminů A/D/E/K).
- **Doporučené pásmo:** 25–35 % celkového kalorického příjmu.
- `1 g tuku = 9 kcal`

### Sacharidy (vyplňující)

```
sacharidy_g = (cílové_kcal − protein_g × 4 − fat_g × 9) / 4
```

`1 g sacharidů = 4 kcal`

### Vláknina

- Doporučeno **25–35 g/den**.
- Pravidlo palce: ~14 g na každých 1000 kcal.

## Příklad výpočtu

Muž, 32 let, 182 cm, 84 kg, aktivní, hubnout mírně:

1. BMR = (10 × 84) + (6.25 × 182) − (5 × 32) + 5 = 840 + 1138 − 160 + 5 = **1823 kcal**
2. TDEE = 1823 × 1.55 = **2826 kcal**
3. Cíl = 2826 − 450 = **2376 kcal**
4. Bílkoviny = 84 × 2.0 = **168 g** = 672 kcal
5. Tuky = 84 × 1.0 = **84 g** = 756 kcal (~32 %)
6. Sacharidy = (2376 − 672 − 756) / 4 = **237 g**
7. Vláknina = ~33 g/den

## Vyvažování v jídelníčku

### Tolerance

| Granularita | Tolerance |
|-------------|-----------|
| Denní součet | ±10 % cíle |
| Jednotlivé jídlo | ±20 % očekávaného podílu |
| Týdenní průměr | ±5 % cíle |

Bílkoviny jsou nejcitlivější — neměly by **klesnout** pod -10 %, ale lehké překročení (+15 %) je OK (zvlášť při hubnutí).

### Rozdělení do jídel

Default rozložení dne (pokud uživatel neřekne jinak):

| Chod | % denních kcal |
|------|----------------|
| Snídaně | 25 % |
| Svačina | 10 % |
| Oběd | 35 % |
| Večeře | 30 % |

Pokud osoba nejí nějaký chod, redistribuuj proporcionálně mezi zbylé.

### Mikro-úpravy

Pokud denní makra nesedí, použij **postup od velkých změn k malým**:

1. **Bílkoviny chybí** → přidej maso/tvaroh/luštěniny do oběda nebo večeře (nejvyšší pohodlí).
2. **Bílkoviny přebývají** → mírně sniž porci masa, nebo přesuň do dalšího dne.
3. **Tuky chybí** → přidej oříšky / olivový olej / avokádo.
4. **Sacharidy chybí** → navyš přílohu (rýže, brambory, ovoce).
5. **Sacharidy přebývají** → sniž přílohu, ne hlavní jídlo.
6. **Kalorie přebývají** → vyřaď snack, nebo zmenši přílohu.

### Multi-person scénář

Když více osob jí stejné jídlo, ale mají různá makra:

**Strategie A — společné vaření, různé porce.** Vař jednou, navaž do krabiček v různých množstvích. Recept zaznamenává **gramáže pro každou osobu zvlášť**.

**Strategie B — sdílený základ + osobní příloha.** Hlavní složku (kuře, omáčka) máte stejnou, ale příloha se liší (Filip: 200 g rýže; Lucka: 80 g rýže + 200 g salátu).

**Strategie C — úplně jiná jídla.** Když se makra liší extrémně (jeden bulk, druhý cut), generuj zvlášť.

V `/jidelnicek` se zeptej, kterou strategii preferuje. Default = B (rozumný kompromis).

## Sanity checks

Před uložením jídelníčku ověř:

- **Žádný den nemá < 80 % cílových kalorií.**
- **Žádný den nemá > 130 % cílových kalorií.**
- **Bílkoviny ≥ 1.5 g/kg každý den.**
- **Tuky ≥ 0.7 g/kg každý den.**
- **Vláknina ≥ 20 g každý den.**

Pokud něco selže, oznámi uživateli a navrhni úpravu, nepokračuj automaticky.
