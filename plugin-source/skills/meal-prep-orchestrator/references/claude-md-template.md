# CLAUDE.md — managed sekce pro meal-prep-rohlik

Tento dokument popisuje, jak orchestrátor a workflow skilly **udržují** sekci v `CLAUDE.md` v root pracovní složky uživatele.

## Pravidla

1. Sekce je vždy ohraničená:
   ```
   <!-- meal-prep-rohlik:start -->
   ... obsah ...
   <!-- meal-prep-rohlik:end -->
   ```

2. **Nikdy nesahej** na obsah mimo tyto značky.

3. Pokud `CLAUDE.md` neexistuje — vytvoř ho s touto sekcí.

4. Pokud existuje a sekce v něm **není** — připoj sekci na konec souboru, předem prázdný řádek.

5. Pokud existuje a sekce v něm **je** — přepiš jen obsah mezi značkami.

## Bash helper

```bash
CLAUDE_MD="./CLAUDE.md"
START_MARKER="<!-- meal-prep-rohlik:start -->"
END_MARKER="<!-- meal-prep-rohlik:end -->"

# Vygeneruj nový obsah do souboru .krabickova-dieta/.claude-md-section.tmp
cat > .krabickova-dieta/.claude-md-section.tmp << 'EOF'
<!-- meal-prep-rohlik:start -->
{managed obsah, viz template níže}
<!-- meal-prep-rohlik:end -->
EOF

if [ ! -f "$CLAUDE_MD" ]; then
  cp .krabickova-dieta/.claude-md-section.tmp "$CLAUDE_MD"
elif grep -q "$START_MARKER" "$CLAUDE_MD"; then
  # Nahraď existující sekci (pomocí python3 — sed je křehký na multiline)
  python3 -c "
import re, sys
with open('$CLAUDE_MD') as f: content = f.read()
with open('.krabickova-dieta/.claude-md-section.tmp') as f: new = f.read().rstrip()
pattern = re.compile(r'<!-- meal-prep-rohlik:start -->.*?<!-- meal-prep-rohlik:end -->', re.DOTALL)
content = pattern.sub(new, content)
with open('$CLAUDE_MD', 'w') as f: f.write(content)
"
else
  # Připoj na konec
  printf '\n' >> "$CLAUDE_MD"
  cat .krabickova-dieta/.claude-md-section.tmp >> "$CLAUDE_MD"
fi

rm .krabickova-dieta/.claude-md-section.tmp
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
