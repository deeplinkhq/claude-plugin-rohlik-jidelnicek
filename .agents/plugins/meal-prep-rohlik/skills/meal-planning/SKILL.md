---
name: meal-planning
description: Best practices a vzory pro sestavování krabičkového jídelníčku — typické chody, doby přípravy, skladovatelnost, příprava na zásobu. Použij při návrhu jídelníčku, výběru receptů a poradenství kolem meal-prep workflow.
---

# Meal Planning — krabičkový workflow

## Co je „krabičková dieta"

Vaření jednou (nebo dvakrát) týdně několika receptů ve větších várkách, rozdělení do krabiček a konzumace 3–7 dní. Cílem je úspora času, kontrola maker, méně rozhodovací únavy.

## Krabičkové vs. ne-krabičkové recepty

### Vhodné

- **Tepelně upravitelné, dobře se ohřívají:** kuře/krůta na různé způsoby, hovězí dušené, mleté maso, ryby (max 2 dny), tofu, tempeh.
- **Stabilní přílohy:** rýže, bulgur, jáhly, quinoa, pohanka, brambory, batáty (vařené i pečené), těstoviny (al dente, mírně podvařené).
- **Dušená/pečená zelenina:** brokolice, květák, papriky, cuketa, mrkev, dýně, růžičková kapusta.
- **Polévky:** kuřecí, hovězí, čočková, dýňová, gulášová.
- **Salát-bowls:** základ obilovina + bílkovina + zelenina + dresink (dresink **odděleně**).

### Nevhodné (špatně se ohřívají / drží)

- Smažené v trojobalu (zvlhne).
- Listové saláty s teplým komponentem (zvadnou).
- Plody moře > 2 dny (kromě konzervovaných).
- Jemné omáčky s vejcem (hollandaise, carbonara) — riziko.
- Křupavé těsto / obalovaná zelenina.

## Doba přípravy a skladovatelnost

| Kategorie | V lednici | V mrazáku |
|-----------|-----------|-----------|
| Vařené maso (kuře/krůta) | 3–4 dny | **nemrazit** |
| Vařené mleté maso | 3–4 dny | **nemrazit** |
| Vařené ryby | **2 dny** | **nemrazit** |
| Vařené luštěniny | 4–5 dní | 6 měsíců |
| Vařená rýže/těstoviny | 3–5 dní | 1 měsíc |
| Pečená/dušená zelenina | 3–4 dny | 2–3 měsíce |
| Polévky bez masa | 3–4 dny | 3 měsíce |
| Polévky s masem | 3–4 dny | **nemrazit** |
| Salát-bowl bez dresinku | 2–3 dny | nemrazit |

> **Pravidlo domácnosti:** maso, drůbež a ryby se v tomto pluginu **nemrazí** — vždy se musí celé spotřebovat v rámci jídelníčku. Pokud balení neodpovídá potřebě, plugin upraví porce nebo přidá další recept (viz skill `waste-minimization` → Hard constraints). Mraženo může být jen vařené luštěniny, přílohy, zelenina a polévka *bez* masa.

**Pravidlo palce pro 5–7denní meal-prep:** ryby plánuj na první 2 dny týdne (krátká skladovatelnost), maso na 3.–4. den. Přílohy, polévky a dušenou zeleninu lze vařit jednou na celý týden a část mrazit, pokud je to potřeba.

## Workflow

### Co a kdy uvařit (pro 5 dnů, 1 osoba, 3 jídla denně)

**Příklad batch-cookingu (cca 2.5 h v neděli):**

1. **Pečicí trouba** (60 min): 1.5 kg kuřete na plechu + 1 kg pečené zeleniny (dvě plechy).
2. **Hrnec 1** (30 min): 800 g rýže.
3. **Hrnec 2** (50 min): 4 l polévka (např. čočková nebo gulášová).
4. **Pánev** (15 min): cibule + paprika + mleté maso na chilli (na 2 dny obědy).
5. **Mixer / shaker:** ovesná kaše-suchý mix do 5 sklenic; přidá se mléko ráno.

Snídaně typu „granola + jogurt" nebo „omeleta s chlebem" — připravit *ráno* (rychlé, max 10 min).

## Časté chyby

- **Příliš málo bílkovin** v krabičkách → nesytí, hlad odpoledne. Cílit na ≥ 30 g B na jídlo.
- **Suché jídlo** → vždy přidat trochu omáčky / olivový olej / dresink (odděleně, aby nenamokal salát).
- **Nudné opakování** → variantnost přes koření a kombinace, ne nutně přes hlavní suroviny. 3 různá koření na kuřeti = 3 různá jídla.
- **Přeplněné krabičky** → ⅓ bílkovina, ⅓ příloha, ⅓ zelenina. Vlhká strana k jedné zdi.
- **Riskantní reheat** → ryby max 2 dny, vejce neohřívat opakovaně.

## Reference soubory

- `references/pantry-staples.json` — výchozí seznam pantry, který má uživatel typicky doma.
- `references/defaults.json` — defaultní `settings.json` při prvním běhu.
- `references/meal-templates.md` — přehled univerzálních „šablon" jídel pro rychlé návrhy.
