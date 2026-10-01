# Bot: sentiment komentátorů na X k SPX

Tento dokument popisuje pouze bota, který sbírá a shrnuje názory komentátorů na X (Twitter) o směru indexu S&P 500 (SPX / futures ES).

## Účel

Bot shrnuje, co si vybraní komentátoři na X myslí o vývoji SPX **dnes a v příštích 1–3 seancích**. Výstup je jen přehled cizích názorů, **nejde o investiční doporučení** ani o vlastní předpověď bota.

## Výstupní soubory

- `data/sentiment.json` – aktuální výsledek (formát popisuje `schemas/sentiment.schema.json`).
- `data/history/sentiment-YYYY-MM-DD.json` – denní archiv; jeden soubor za každý obchodní den.

## Rozvrh

- Pondělí až pátek ve **14:30** a **15:15** pražského času. Přechod mezi zimním (SEČ) a letním (SELČ) časem se řeší automaticky.
- Druhý běh (15:15) přepisuje výsledek prvního běhu, a to v `data/sentiment.json` i v archivu daného dne.
- Vše musí být hotovo a zapsáno nejpozději v **15:35** pražského času.

## Zdroje

- Seznam sledovaných účtů je v `config/x_sources.json`. Účty vybral bot na pokyn uživatele dne 2026-10-01.
- Seznam je zmrazen do **2026-10-31**. Potom se smí měnit nejvýše jednou měsíčně, a to jen kvůli náhradě smazaného, suspendovaného nebo zjevně nekvalitního účtu.
- Každá změna se zapisuje s datem a důvodem do `config/x_sources_changelog.md`.

## Metoda

1. Bot načte příspěvky sledovaných účtů za **posledních 24 hodin**. Příspěvky z **posledních 6 hodin** mají vyšší váhu.
2. Započítávají se jen příspěvky, které **výslovně** vyjadřují názor na směr SPX nebo ES. Obecné komentáře k ekonomice nebo jednotlivým akciím se ignorují.
3. Každý započítaný příspěvek dostane postoj: býčí (`bullish`), medvědí (`bearish`) nebo neutrální (`neutral`).
4. Pokud se směrově vyjádří méně než **5 účtů**, je výsledek `status: "partial"` a `direction: "insufficient"`.
5. Pokud jsou názory výrazně rozdělené, je směr `mixed`.
6. Skóre se počítá jako **(býčí − medvědí) / celkem × 2** a zaokrouhluje se na jedno desetinné místo, takže leží v rozsahu −2 až 2.
7. Účty se střetem zájmů (`conflict_of_interest: true`, typicky prodej předplatného nebo signálů) mají váhu **0,5**, ostatní 1,0.

## Pole `direction_cs` a `one_liner_cs`

Obě pole jsou povinná a slouží k rychlému přečtení výsledku v češtině. **Věty jen shrnují, co píšou sledovaní komentátoři na X. Nejsou to předpovědi bota ani investiční doporučení.**

**`direction_cs`** – směr česky, odvozený z pole `direction`:

| `direction` | `direction_cs` |
|---|---|
| `bullish` | nahoru |
| `bearish` | dolů |
| `neutral` | neutrálně |
| `mixed` | nejasné |
| `insufficient` | nejasné |

**`one_liner_cs`** – přesně jedna česká věta:

- Pokud `status` není `ok`, je věta vždy přesně: „Nedostatek dat pro sentiment.“
- Pokud je `status` `ok`, věta závisí na směru:
  - `bullish`: „SPX půjde nahoru (x z y účtů, důvěra …).“
  - `bearish`: „SPX půjde dolů (x z y účtů, důvěra …).“
  - `neutral`: „SPX bude spíš v pohybu do strany (x z y účtů, důvěra …).“
  - `mixed`: „Názory na SPX jsou rozdělené (x z y účtů, důvěra …).“

Přitom `y` = `accounts_considered` a `x` = počet účtů s vítězným postojem (u `mixed` počet účtů v největší skupině). Slovo důvěry odpovídá poli `confidence`: `low` → nízká, `medium` → střední, `high` → vysoká.

Příklad: „SPX půjde nahoru (8 z 12 účtů, důvěra střední).“ znamená, že 8 z 12 vyhodnocených komentátorů psalo býčí názor. Neznamená to, že SPX skutečně poroste.

Schéma `schemas/sentiment.schema.json` hlídá mapování směru, tvar věty, slovo důvěry a pravidlo pro `status` jiný než `ok`. Nekontroluje, zda čísla `x` a `y` odpovídají datům; to musí zajistit bot.

## Význam stavů (`status`)

- `ok` – dost dat, výsledek je úplný.
- `partial` – neúplná data (např. méně než 5 účtů se směrovým názorem nebo část účtů nešla načíst); směr bývá `insufficient`.
- `unavailable` – data nebyla k dispozici vůbec (např. chyba nebo limit X API, chybějící přístup).

## Nastavení

a) **Přístup k X API:** Bearer token se zadává **pouze** do zabezpečeného pole pro tajné údaje (secrets) bota. Nikdy ho nevkládejte do souborů v repozitáři, do chatu ani do logů.

b) **Přístup ke GitHubu:** přes připojený GitHub účet (konektor) s právem zápisu do tohoto repozitáře. Bot zapisuje **jen své vlastní soubory** (`data/sentiment.json` a `data/history/sentiment-*.json`) a na nic jiného nesahá. Commity mají zprávu ve tvaru `bot: x-sentiment <ISO timestamp>`.

c) **Konflikty při commitu:** pokud zápis selže kvůli konfliktu, bot to zkusí znovu nejvýše **3×**. Pak selhání nahlásí a ostatní soubory nechá beze změny.

## Omezení

- Track record a přesnost předpovědí sledovaných komentátorů **nejsou ověřeny**.
- X API má limity počtu požadavků a je placené. Při vyčerpání limitu může být výsledek `partial` nebo `unavailable`.
- Bez přístupu k API nelze zjistit, zda byl některý účet smazán nebo suspendován.
- Klasifikace postoje je automatická a může se mýlit, hlavně u ironie nebo podmíněných výroků.
