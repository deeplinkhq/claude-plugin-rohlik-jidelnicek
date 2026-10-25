# AGENTS.md — managed sekce pro meal-prep-rohlik

Tento dokument popisuje, jak orchestrátor a workflow skilly **udržují** sekci v `AGENTS.md` v root pracovní složky uživatele.

## Pravidla

1. Sekce je vždy ohraničená:
   ```
   <!-- meal-prep-rohlik:start -->
   ... obsah ...
   <!-- meal-prep-rohlik:end -->
   ```

2. **Nikdy nesahej** na obsah mimo tyto značky.

3. Před vytvořením souboru zkontroluj starou instalaci: pokud je managed sekce jen v `CLAUDE.md` a `AGENTS.md` neexistuje, přejmenuj celý původní soubor na `AGENTS.md` beze změny ostatních pravidel. Pokud existují oba, starý soubor nemá managed značky nebo je některý instrukční soubor symlink, zastav se s konkrétním požadavkem na sloučení instrukcí; nikdy nevytvářej druhou nezávislou kopii. Když není stará instalace ani `AGENTS.md`, vytvoř ho s touto sekcí.

4. Pokud existuje a sekce v něm **není** — připoj sekci na konec souboru, předem prázdný řádek.

5. Pokud existuje a sekce v něm **je** — přepiš jen obsah mezi značkami.

## Bash helper

```bash
AGENT_INSTRUCTIONS="./AGENTS.md"
START_MARKER="<!-- meal-prep-rohlik:start -->"
END_MARKER="<!-- meal-prep-rohlik:end -->"

LEGACY_INSTRUCTIONS="./CLAUDE.md"
if [ -L "$AGENT_INSTRUCTIONS" ]; then
  echo "AGENTS.md je symlink. Nejprve určete kanonický soubor pro sloučení instrukcí." >&2
  exit 1
fi
if [ -e "$LEGACY_INSTRUCTIONS" ] || [ -L "$LEGACY_INSTRUCTIONS" ]; then
  if [ -L "$LEGACY_INSTRUCTIONS" ] || [ ! -f "$LEGACY_INSTRUCTIONS" ] || [ -e "$AGENT_INSTRUCTIONS" ] || ! grep -q "$START_MARKER" "$LEGACY_INSTRUCTIONS" || ! grep -q "$END_MARKER" "$LEGACY_INSTRUCTIONS"; then
    echo "Nejprve slučte staré CLAUDE.md do AGENTS.md se zachováním projektových pravidel; profil zatím nebyl změněn." >&2
    exit 1
  fi
  mv "$LEGACY_INSTRUCTIONS" "$AGENT_INSTRUCTIONS"
fi

# Vygeneruj nový obsah do souboru .krabickova-dieta/.agents-md-section.tmp
cat > .krabickova-dieta/.agents-md-section.tmp << 'EOF'
<!-- meal-prep-rohlik:start -->
{managed obsah, viz template níže}
<!-- meal-prep-rohlik:end -->
EOF

if [ ! -f "$AGENT_INSTRUCTIONS" ]; then
  cp .krabickova-dieta/.agents-md-section.tmp "$AGENT_INSTRUCTIONS"
elif grep -q "$START_MARKER" "$AGENT_INSTRUCTIONS"; then
  # Nahraď existující sekci (pomocí python3 — sed je křehký na multiline)
  python3 -c "
import re, sys
with open('$AGENT_INSTRUCTIONS') as f: content = f.read()
with open('.krabickova-dieta/.agents-md-section.tmp') as f: new = f.read().rstrip()
pattern = re.compile(r'<!-- meal-prep-rohlik:start -->.*?<!-- meal-prep-rohlik:end -->', re.DOTALL)
content = pattern.sub(new, content)
with open('$AGENT_INSTRUCTIONS', 'w') as f: f.write(content)
"
else
  # Připoj na konec
  printf '\n' >> "$AGENT_INSTRUCTIONS"
  cat .krabickova-dieta/.agents-md-section.tmp >> "$AGENT_INSTRUCTIONS"
fi

rm .krabickova-dieta/.agents-md-section.tmp
```

## Šablona obsahu sekce

```markdown
<!-- meal-prep-rohlik:start -->
## Krabičková dieta — kontext projektu

Plugin **meal-prep-rohlik** spravuje v této složce krabičkový jídelníček, makra a nákupní seznamy z Rohlik.cz. Veškerá komunikace probíhá **česky**.

### Aktivní profily osob

| Jméno | Cíl | Denní kcal | B / S / T (g) | Soubor |
|-------|-----|------------|---------------|--------|
| {Filip} | {hubnout mírně} | {2376} | {168 / 230 / 79} | `.krabickova-dieta/profiles/filip.json` |
| {Lucka} | {udržovat} | {1850} | {110 / 200 / 65} | `.krabickova-dieta/profiles/lucka.json` |

### Nastavení

- **Max cena/den/osoba:** {250} Kč
- **Strategie odpadu:** {trvanlivé potraviny}
- **Pantry:** {seznam zkráceně, max 6 položek + „...nebo viz settings.json"}

### Stav

- **Poslední jídelníček:** {2026-05-11, 5 dní, soubor `.krabickova-dieta/jidelnicky/2026-05-11.md`} _(nebo „zatím žádný")_
- **Cache produktů:** {47 položek, nejstarší cena z 2026-05-09} _(nebo „prázdná")_
- **Cache receptů:** {12 receptů, top: kuřecí stripsy s rýží 5×}

### Jak se to ovládá

Stačí napsat přirozeně, co chceš — plugin sám pozná, jestli jde o nastavení maker, plánování jídelníčku, nákup, nebo otázku na výpočet. Slash command je jen jeden:

- `/cache-info` — správa lokální cache (refresh cen, mazání, export)

### Pravidla pro Claude

- Nutriční data a ceny **pouze z rohlik.cz** přes MCP `rohlik-official`.
- **Cache-first** — vždy se podívej do `.krabickova-dieta/cache/products.json` před voláním MCP.
- **Modifikační akce na Rohlíku** (přidat do košíku, vytvořit shopping list, smazat cache) **vždy s explicitním potvrzením v chatu**.
- **Maso, drůbež a ryby se nemrazí** — musí být celé spotřebovány v rámci jídelníčku. Pokud balení neodpovídá potřebě, uprav porce nebo přidej recept (viz `waste-minimization` Hard constraints).
- Při generování jídelníčku používej skill `meal-planning`, pro vážení maker `macro-balancing`, pro odpad `waste-minimization`, pro produkty `rohlik-product-lookup`.
<!-- meal-prep-rohlik:end -->
```

## Kdy aktualizovat

| Akce | Aktualizovat sekci? |
|------|----------------------|
| Vytvořen nový profil v `setup-macro-profile` | ✓ |
| Změna existujícího profilu | ✓ |
| Smazání profilu | ✓ |
| Vygenerován nový jídelníček | ✓ |
| Vytvořen nový nákupní seznam | ✓ |
| Změna `settings.json` | ✓ |
| Refresh cache (jen ceny) | jen pokud se výrazně změnil počet stale položek |
| Volání čistých výpočetních skillů (`macro-balancing` na otázku) | ✗ |
