---
name: setup-macro-profile
description: Vytváří, upravuje nebo maže makro profily osob pro krabičkovou dietu. Aktivuj, když uživatel chce nastavit makra, spočítat denní příjem, vytvořit profil, přidat další osobu do plánování, změnit váhu/cíl/aktivitu, nebo když chybí profil potřebný pro jiný workflow. Po dokončení aktualizuje CLAUDE.md.
---

# Setup Macro Profile

Workflow pro správu profilů osob — výpočet BMR/TDEE, cílových kalorií a rozložení makronutrientů. Používá vzorce ze skillu `macro-balancing`. Veškerá komunikace **česky**.

Profily se ukládají do **pracovní složky uživatele** (cwd), ne do plugin složky.

## Krok 0: Připrav úložiště a načti stav

```bash
mkdir -p .krabickova-dieta/profiles
ls .krabickova-dieta/profiles/*.json 2>/dev/null
```

Vypiš seznam existujících profilů (jméno + cíl + denní kcal) ze souborů `.krabickova-dieta/profiles/*.json`.

## Krok 1: Zjisti, co uživatel chce

Pomocí AskUserQuestion (jen pokud z konverzace není zřejmé):

- **Co budeme dělat?**
  - Přidat novou osobu
  - Upravit existující profil ({jméno})
  - Smazat profil
  - Jen se podívat na aktuální nastavení

Pokud z předchozí konverzace víš (např. orchestrátor tě sem poslal s tím, že profil chybí pro plánování) — přeskoč otázku a rovnou navrhni vytvoření profilu pro hlavního uživatele.

## Krok 2: Sběr údajů (přidat / upravit)

Max 4 otázky najednou.

**Kolo 1 — základ:**
- **Jméno osoby** (volný text — slouží jako klíč i název souboru, převeď na kebab-case)
- **Pohlaví:** muž / žena
- **Věk** (let)
- **Výška** (cm)

**Kolo 2 — váha a aktivita:**
- **Aktuální váha** (kg)
- **Cílová váha** (kg, volitelné)
- **Úroveň aktivity:** sedavá / mírně aktivní / aktivní / velmi aktivní / extrémně aktivní
- **Cíl:** hubnout (mírně) / hubnout (agresivně) / udržovat / nabírat lean / nabírat (bulk)

**Kolo 3 — preference:**
- **Dietní preference:** bez omezení / bezlepková / bezlaktózová / vegetariánská / veganská (multiSelect: true)
- **Alergie nebo zakázané potraviny** (volný text, volitelné)
- **Které chody osoba jí?** snídaně / svačina / oběd / večeře (multiSelect: true)

## Krok 3: Vypočítej makra

Použij vzorce z `@${CLAUDE_PLUGIN_ROOT}/skills/macro-balancing/SKILL.md`:
1. BMR (Mifflin-St Jeor)
2. TDEE = BMR × faktor aktivity
3. Cíl = TDEE ± deficit/surplus
4. Bílkoviny → tuky → sacharidy (vyplňující)

Stručně rekapituluj uživateli a zeptej se, jestli chce ručně doladit.

**Bezpečnostní limity** (z `macro-balancing`): žádné kcal pod 1200 (žena) / 1500 (muž), při BMI < 18.5 nedoporučovat hubnutí.

## Krok 4: První profil → globální settings

Pokud `.krabickova-dieta/settings.json` ještě neexistuje, doplň globální nastavení (jednou):

- **Maximální cena/den/osobu** (Kč, default 250)
- **Pantry** — použít výchozí seznam ze `${CLAUDE_PLUGIN_ROOT}/skills/meal-planning/references/pantry-staples.json` / vybrat ručně
- **Strategie odpadu:** trvanlivé potraviny / levné potraviny / absolutní minimum

## Krok 5: Ulož profil

`.krabickova-dieta/profiles/{jmeno-kebab-case}.json`:

```json
{
  "name": "Filip",
  "created": "2026-05-11",
  "updated": "2026-05-11",
  "profile": {
    "gender": "muž",
    "age": 32,
    "height_cm": 182,
    "weight_kg": 84,
    "target_weight_kg": 78,
    "activity_level": "aktivní",
    "goal": "hubnout (mírně)",
    "dietary_preferences": ["bez omezení"],
    "forbidden": [],
    "meals": ["snídaně", "oběd", "večeře"]
  },
  "calculated": {
    "bmr_kcal": 1823,
    "tdee_kcal": 2826,
    "target_kcal": 2376
  },
  "macros": {
    "protein_g": 168,
    "carbs_g": 237,
    "fat_g": 84,
    "fiber_g": 33
  }
}
```

A `settings.json` (jen při prvním běhu):

```json
{
  "max_price_per_day_per_person_czk": 250,
  "pantry": ["Sůl", "Pepř", "Olivový olej", "..."],
  "waste_strategy": "trvanlivé potraviny",
  "currency": "CZK"
}
```

## Krok 6: Aktualizuj CLAUDE.md

**Vždy** po vytvoření / úpravě / smazání profilu aktualizuj managed sekci v `./CLAUDE.md` v projektu. Postup je v `@${CLAUDE_PLUGIN_ROOT}/skills/meal-prep-orchestrator/references/claude-md-template.md`. Pokud `CLAUDE.md` neexistuje, vytvoř ho.

## Krok 7: Potvrď a pokračuj

Stručně shrň, co bylo uloženo. Pokud byl tento skill spuštěn jako **dependency** pro jiný workflow (např. orchestrátor tě sem poslal před `generate-meal-plan`), **automaticky pokračuj** do toho workflow — uživatel by neměl muset opakovat původní žádost.

Pokud byl spuštěn samostatně, řekni: *„Hotovo. Stačí napsat 'naplánuj jídelníček' a sestavím ho podle nového profilu."*

## Smazání profilu

Pokud uživatel chce smazat profil:
1. Vypiš, který soubor se chystáš smazat (`.krabickova-dieta/profiles/{jmeno}.json`).
2. **Vyžaduj explicitní potvrzení** v chatu („Smazat profil {jméno}?").
3. Po potvrzení použij `rm` přes Bash.
4. Aktualizuj CLAUDE.md.

## Důležité zásady

- **Vše ukládej do `./.krabickova-dieta/`** v pracovní složce uživatele.
- **Komunikuj česky.**
- **Bezpečnostní limity** dodržuj striktně.
- **CLAUDE.md aktualizuj vždy** po změně profilu/settings.
- **Mazání vyžaduje potvrzení** v chatu.
