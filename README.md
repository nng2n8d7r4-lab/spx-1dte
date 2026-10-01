# SPX 1DTE kalkulačka

Denní rozhodovací pomůcka pro kreditní spready SPX s expirací příští seanci (bear call, bull put, iron condor). Není to investiční doporučení. Prahy jsou výchozí a mají se kalibrovat podle deníku.

## Jak to otevřít

Aplikace je jeden soubor [site/index.html](site/index.html) a tři JSON vedle něj ve [site/data/](site/data/). Žádný build, žádné CDN.

GitHub Pages:

1. V repozitáři nech složku `site` tak, jak je, nebo zkopíruj její obsah (`index.html` + `data/`) do kořene větve `gh-pages`.
2. V nastavení Pages zvol tuto větev a kořen `/` (nebo `/site`, když Pages publikuje z hlavní větve a aplikace zůstane ve složce `site`).
3. Otevři stránku. Tlačítko **Načíst data** stáhne `./data/market.json`, `./data/calendar.json` a `./data/sentiment.json` s `cache: no-store`.

Relativní cesty musí zůstat: `index.html` a `data/` vedle sebe.

## Odkud jsou tržní data (režim B)

Prohlížeč **nemůže** volat konektor IBKR. Konektor je dostupný jen agentovi / botovi, ne z frontendu statické stránky. Tlačítko proto načítá už hotový `market.json`.

Bot (jen čtení trhu) má:

1. Najít ES přes vyhledání futures, vzít nejbližší nesplacený kontrakt (nic natvrdo nepsat, v prosinci se rolují).
2. Vzít SPX, VIX a VIX1D podle symbolu, ne podle natvrdo daného čísla kontraktu.
3. Denní úsečky SPX (RTH). Úsečku s datem ≥ dnešní den v `America/New_York` zahodit.
4. ATM IV: nejbližší expirace SPXW **po** dnešním newyorském datu, call nejblíž odhadovanému spotu, pole mid IV annualizované v procentech.
5. Zapsat `data/market.json`. Když jedno volání selže, ostatní pole stejně uložit.

Povolené je jen čtení trhu: vyhledání kontraktu, snapshot, historie, parametry opcí a řetěz. Zakázané je cokoli o účtu, příkazech, pokynech, alertech nebo seznamech sledovaných instrumentů.

Účet v době sestavení neměl živé předplatné. Ceny v ukázkovém `market.json` jsou zpožděné asi o 15 minut (snímek 1. 10. 2026 ráno SEČ, SPX i VIX1D byly mimo cash seanci zmrazené). V rozhraní je u nich štítek „zpožděná data“ a čas v SEČ/SELČ. Nepoužívej je jako živou cenu.

Kalendář a sentiment konektor nedává. V repu jsou ukázkové soubory. Oranžové kolonky (delta, zmrazený VIX1D, makro pozadí, cokoli co se nenačetlo) vyplň ručně.

Když `market.json` obsahuje `status: "auth_expired"` nebo `"not_connected"`, stránka ukáže příslušnou českou hlášku a nic nespočítá z prázdna.

## Schéma `data/market.json`

| pole | význam |
| --- | --- |
| generated_at | ISO čas snímku |
| status | `ok`, `auth_expired`, `not_connected` |
| delayed | true, dokud jde o zpožděná data |
| es_last, es_prior_close | poslední a včerejší zavření nejbližšího kontraktu ES |
| es_contract_month | `YYYYMM`, jen popis |
| es_status, es_ts | stav a čas kotace |
| spx_last, spx_is_frozen | index; true když je zavírací / FROZEN |
| vix_last, vix_prior_close | VIX |
| vix_prior_close_derived | true, když včerejšek vznikl jako last − change |
| vix1d_last, vix1d_is_frozen | když frozen, faktor 2 zůstane prázdný |
| atm_iv_pct | roční IV v procentech |
| expiry_date | `YYYY-MM-DD` použité SPXW |
| iv_strike, iv_trading_class | jaký kontrakt dal IV |
| daily_bars | `{date, open, high, low, close}`, nejstarší první, jen uzavřené seance |

Spot: když `spx_is_frozen`, `spx_last × (1 + es_změna/100)`, jinak `spx_last`. Změna ES = `es_last / es_prior_close − 1`. V 15:45 SEČ to není čistý noční gap.

## Schéma `data/calendar.json`

`generated_at`, `status` (`ok`), `sample`, `macro_today`, `macro_tomorrow`, `megacap_earnings_today`, `shortened_session`, `events[]` s `name` a `name_cs`.

Když je soubor z dnešního pražského dne, ne starší než 10 hodin, `status` je `ok`, `cross_checked` je `true`, `today.date` je dnešní den v `America/New_York` a `big_macro_today` i `big_macro_tomorrow` jsou boolean, aplikace sama zaškrtne potvrzení (příznak false) nebo stopku (příznak true). Jinak se nesaškrtne nic a potvrzení zůstanou oranžová. Earnings a zkrácená seance se doplní jen když je jejich příznak boolean, jinak zůstanou oranžové. Skóre se nemění.

## Schéma `data/sentiment.json`

`generated_at`, `status`, `direction` (`bullish` / `bearish` / `neutral`), `score`, `confidence`, `summary`, `caveats[]`, `posts[]` (`handle`, `stance`, `link`, `text`).

Starší než 24 h nebo `status` jiné než `ok` → „bez sentimentu“. Do skóre, sklonu ani strategie se sentiment nepočítá. Do deníku se uloží směr a skóre, aby šel později ověřit.

## Skóre a sklon

Sedm faktorů 0–2, maximum 14. Čtyři signály sklonu −1/0/+1. Prahy neměň bez domluvy; v aplikaci jsou pod každým faktorem šedě.

Verdikt: 11–14 Obchodovat (šířka 15–20), 7–10 Opatrně (šířka 10), 0–6 Vynechat. Stopka → Neobchodovat. Bez obou potvrzení kalendáře → Zkontroluj kalendář.

Odhad strike: `sg = IV/100 × √(1/252)`, `z = inverzní normální kvantil(1 − delta/100)`, call `S × exp(z·sg + sg²/2)`, put `S × exp(−z·sg + sg²/2)`, zaokrouhlení na 5. Jeden ATM IV ignoruje skew.

## Deník

`localStorage` v prohlížeči, klíč `spx1dte-journal-v1`, nejvýše 60 záznamů. Stav uložení je pod tlačítkem. Smazání chce druhé kliknutí. Export je středníkem oddělený text, desetinná čárka, hlavička anglicky.

Vyhodnocení bere první uzavřenou úsečku s datem po dni vstupu (den vstupu je datum v `America/New_York`). Směr: býčí když close > spot, medvědí když close < spot, neutrální když je close v očekávaném pohybu. Bear call vyhrává když close < short call, bull put když close > short put, iron condor když je close mezi nimi. „Nevstoupím“ se do výher strategie nepočítá.

## Testy

Vektorové kontroly jsou v `window.SPX.selfCheck()` a `window.SPX.applyConnector()` (simulace konektoru, stránka sama konektor nevolá).

```bash
node scripts/spx-qa.mjs http://127.0.0.1:8080/site/index.html
```

Skript čeká, že stránka běží. Ověřuje strike 7 795 / 7 610 / ±73, gap −0,55 %, VIX 16,43, zmrazený spot, verdikt bez kalendáře, stopku, pád chybějícího JSON, pád samotného VIX1D a vyhození dnešní nedokončené úsečky.
