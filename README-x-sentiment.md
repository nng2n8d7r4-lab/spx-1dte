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
