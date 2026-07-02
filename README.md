# ELKO Lagerleiðrétting (Lagerlr)

Innra verkfæri fyrir ELKO til að tilkynna og fylgjast með lagerfrávikum (gölluðum vörum, þjófnaði, ósöluhæfum vörum og lagerleiðréttingum) milli allra ELKO verslana.

Byggt á GitHub Pages (framendi), Power Automate (bakendi/vinnuflæði) og SharePoint listum (gagnageymsla), með Cloudflare Worker sem milliliggjandi proxy til að fela viðkvæm gögn.

---

## Yfirlit

- **Almennt eyðublað** (`index.html` / `lagerlr-public.html`) — starfsfólk tilkynnir frávik, velur verslun, leitar að vöru með sjálfvirkri útfyllingu (autocomplete)
- **Stjórnborð** (`admin.html`) — innskráning eftir hlutverki, yfirlit yfir allar tilkynningar, tölfræði, notendastjórnun

---

## Tæknileg uppbygging

```
Vafri (GitHub Pages)
      │
      ▼
Cloudflare Worker (lagerlr-proxy)
      │
      ▼
Power Automate flæði
      │
      ▼
SharePoint listar (festihf.sharepoint.com)
```

### Af hverju proxy?

Upprunalega voru Power Automate vefslóðir (með `sig=` tókenum) og API lyklar geymdir beint í JavaScript kóðanum á síðunum. Þar sem GitHub Pages er opinbert, þýddi það að hver sem er gat skoðað síðukóðann og fundið þessar vefslóðir/lykla — og þar með fengið beinan aðgang að öllum flæðum, algjörlega framhjá innskráningu.

Cloudflare Worker leysir þetta: vafrinn talar aðeins við Worker-inn (t.d. `https://lagerlr-proxy.pruins.workers.dev/api/tilkynningar`), og Worker-inn — sem keyrir á netþjóni, ekki í vafranum — er sá sem raunverulega kallar á Power Automate með réttum vefslóðum og API lykli. Þessi gögn eru geymd sem **leynd breytur (secrets)** í Cloudflare og birtast aldrei í síðukóðanum.

Worker-inn er sameiginlegur milli Lagerlr og Bankinn (annað repo), staðsettur í eigin verkefni (`lagerlr-proxy`) óháð báðum.

---

## SharePoint listar

**`Vörulisti`** — grunnlisti yfir allar vörur (~38.800 færslur)

| Dálkur | Tegund | Lýsing |
|---|---|---|
| Title | Texti | Vörunúmer |
| Vara | Texti | Vöruheiti |
| Deild | Texti | Deild/flokkur |
| EOL | Já/Nei | Er varan hætt (End of Life). 1=Já, 0=Nei í upprunagögnum |
| Uppfært | Dagsetning | Síðast uppfært við innflutning |

**`Tilkynningar`** — allar frávikstilkynningar

| Dálkur | Tegund | Lýsing |
|---|---|---|
| Title | Texti (auto) | LR-yyMMddHHmmss |
| Verslun | Texti | Hvaða ELKO verslun |
| Vörunúmer | Texti | |
| Vara | Texti | |
| Ástæða | Choice | Gallað / Ósöluhæft / Þjófnaður / Lagerleiðrétting |
| Lýsing | Texti | Nánari lýsing |
| Tilkynnir | Texti | Hver tilkynnir |
| Staða | Choice | Ný / Í vinnslu / Lokið |
| Dagsetning | Dagsetning | Hvenær tilkynnt |
| StadaDagsetning | Dagsetning | Síðast uppfærð staða |

**`Notendur`** — innskráningar og hlutverk fyrir stjórnborð

| Dálkur | Tegund | Lýsing |
|---|---|---|
| Title | Texti | Netfang |
| Lykilord | Texti | SHA-256 hash |
| Hlutverk | Choice | Verslunarstjóri / Lagerstjóri / Skrifstofa / Master |
| Verslun | Texti | Aðalverslun |
| ViðbótarVerslun | Multi-choice | Aukaverslanir (t.d. fyrir Verslunarstjóra sem sér fleiri en eina verslun) |
| LykilordSett | Já/Nei | Hefur notandi sett lykilorð |
| ResetKodi | Texti | Endurstillingarkóði |

**Verslanir:** ELKO Lindir, ELKO Smáralind, ELKO Skeifan, ELKO Grandi, ELKO Akureyri, ELKO Flugstöð

---

## Power Automate flæði

Öll flæði eru kölluð í gegnum Cloudflare Worker-inn, aldrei beint úr vafra.

| Route í Worker | Virkni |
|---|---|
| `/vorulisti` | Leitar í Vörulista (fyrir sjálfvirka útfyllingu í almennu formi) |
| `/senda-tilkynningu` | Býr til nýja tilkynningu |
| `/tilkynningar` | Sækir allar tilkynningar |
| `/uppfaera-stodu` | Uppfærir stöðu tilkynningar |
| `/notandi` | Sækir einn notanda (innskráning) |
| `/lykilord` | Setur lykilorð |
| `/nyr-notandi` | Býr til nýjan notanda |
| `/reset` | Sendir endurstillingarkóða |
| `/stadfesta-reset` | Staðfestir endurstillingu lykilorðs |
| `/notendur` | Sækir alla notendur |
| `/uppfaera-notanda` | Uppfærir notanda |
| `/endurstilla-notanda` | Endurstillir lykilorð notanda |
| `/eyda-notanda` | Eyðir notanda |

**Innflutningur á Vörulista** (`ELKO - Flytja inn Vörulista`, keyrt handvirkt, ekki kallað úr vafra):
- Les Excel skjöl úr OneDrive `/Lagerlr_skjal/`, skipt í ~5.000 raða búta (chunk1, chunk2, o.s.frv., hver með Excel Table sem heitir `Data`)
- Fyrir hverja röð: athugar hvort Vörunúmer sé nú þegar til (Get items á Title) → Update ef til, Create ef ekki
- EOL dálkur: `equals(string(items('Apply_to_each_1')?['EOL']), '1')` — breytir 1/0 úr Excel í Já/Nei
- Uppfært dálkur settur með `utcNow()` við hverja keyrslu

---

## Admin panel — virkni

**Innskráning:** SHA-256 lykilorð, fyrsta-innskráning flæði, gleymt lykilorð með 6 stafa kóða sendum í tölvupósti, `sessionStorage` fyrir setu.

**Flipar eftir hlutverki:**
- Tilkynningar (allir)
- Tölfræði (Verslunarstjóri, Lagerstjóri, Skrifstofa, Master)
- Notendur (aðeins Master)

**Tilkynningar flipi:** síur (verslun/staða/ástæða), rauð flögg fyrir endurteknar tilkynningar sömu vöru (🟡 2, 🔴 3+), stöðusamantekt, raðað eftir nýjustu fyrst.

**Tölfræði flipi:** í dag/viku/mánuði/ári, eftir ástæðu, eftir verslun, "Vaxandi vandamál" — vörur með 5+ Gallað eða Þjófnaður tilkynningar síðustu 30 daga, sundurliðað eftir ástæðu.

---

## Cloudflare Worker (`lagerlr-proxy`)

**Staðsetning:** `~/Desktop/lagerlr-proxy` (staðbundið verkefni), keyrt með Wrangler CLI. Sameiginlegur með Bankinn repo.

**Uppfæra kóða:**
```bash
cd lagerlr-proxy
# breyta src/index.js
wrangler deploy
```

**Bæta við nýrri leynd (secret):**
```bash
wrangler secret put HEITI_A_LEYND
```

**Skoða allar leyndir (bara nöfn, ekki gildi):**
```bash
wrangler secret list
```

**Mikilvægt:** Ef nýtt flæði er búið til í Power Automate fyrir Lagerlr, þarf að:
1. Bæta nýrri línu í `FLOW_MAP` í `src/index.js` (t.d. `'nytt-flaedi': 'FLOW_NYTT_FLAEDI'`)
2. Keyra `wrangler deploy`
3. Bæta við nýrri leynd með `wrangler secret put FLOW_NYTT_FLAEDI`
4. Uppfæra viðeigandi HTML skjal (`admin.html` eða almenna formið) til að nota `PROXY + '/nytt-flaedi'`

---

## Öryggi

- Engin API lykill eða Power Automate vefslóð er sýnileg í síðukóðanum á GitHub Pages
- Git-saga þessa repository hefur verið hreinsuð með BFG Repo-Cleaner til að fjarlægja fyrri útgáfur af þessum gögnum
- CORS á Worker-num er læst við `https://miketeex.github.io`, svo aðeins þessi síða getur kallað á hann
- Ef grunur leikur á að einhver leynd hafi lekið (t.d. með óvart commit), skal strax:
  1. Búa til nýja Power Automate HTTP kveikju (breytir `sig=` tókeninu)
  2. Uppfæra samsvarandi leynd í Worker með `wrangler secret put`
  3. Ef þörf er á, endurtaka BFG hreinsun á repository

---

## Þekkt verkefni sem eftir er (TODO)

- **Sjálfvirk samstilling Vörulista** — núna handvirkt flutt inn úr Excel skjölum (skipt í 5.000 raða búta). Uppruni gagna er í Power BI Titan gagnalíkani (ProductNo, ProductName í Product töflu). Þarf lausn sem sneiðir hjá 30k raða útflutningstakmörkun Power BI.
- **Mánaðarleg tölfræðiskýrsla í tölvupósti** — opt-in/opt-out kerfi þar sem notandi stillir sjálfur hvort hann fái skýrslu (nýr `ManadarskyrslaOptIn` dálkur á Notendur, "Mínar stillingar" svæði í admin panel). Verslunarstjóri fær tölfræði fyrir sína verslun, Master/Skrifstofa fá heildarmynd. Áætlað að nota QuickChart.io fyrir myndræn gröf (súlurit eftir verslun, kökurit eftir ástæðu, línurit yfir þróun síðustu 6 mánuði).

---

## Höfundur

Magnús — ELKO, Power Platform / SharePoint þróun.
