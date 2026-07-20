---
name: finalize
description: >-
  Session lopetusrutiini ("finalize"): käy keskustelu läpi, ehdota korkean
  arvon opit hyväksyttäväksi/hylättäväksi ja tallenna hyväksytyt projektin
  muistiin (CLAUDE.md + mahdollinen projektimuistio), sekä auditoi olemassa
  oleva muisti roskan varalta. Käytä AINA kun käyttäjä sanoo /finalize,
  "lopetellaan", "otetaan opit talteen", "wrap up", "finalize the session",
  "päivitä muisti tästä sessiosta" tai vastaavaa session päätteeksi — myös
  silloin kun käyttäjä kiittelee ja on selvästi lopettelemassa isoa
  työrupeamaa ja muistiin kirjaamatonta oppia on kertynyt.
---

# Finalize — session opit talteen

Tarkoitus: vähentää tulevien sessioiden tokenikulua ja toistotyötä. Sama
umpikuja (esim. työkalu joka ei toimi tietyllä tavalla) poltetaan muuten
uudelleen joka sessiossa. Tämä skill kerää sellaiset opit KERRAN ja
tallentaa ne sinne, mistä tuleva sessio ne oikeasti löytää.

## Muistipaikat ja työnjako

- **`CLAUDE.md`** (projektin juuressa; luo jos puuttuu) — latautuu
  automaattisesti joka session alussa, joten JOKAINEN rivi maksaa tokeneita
  ikuisesti. Tänne vain tiivistä, ajatonta "näin tässä projektissa
  työskennellään" -tietoa: build/testikomennot, deploy-polku, ympäristön
  erikoisuudet, työkalujen sudenkuopat kiertoteineen. Yksi rivi per oppi.
  Pidä koko tiedosto alle ~60 rivissä — jos se paisuu, auditoi ennen kuin
  lisäät.
- **Projektin muistio/dokumentaatio** (jos projektilla on sellainen, esim.
  PROJEKTI-*.md, NOTES.md, docs/) — tila ja historia. Tänne laajempi
  konteksti: päätökset perusteluineen, toteutusyksityiskohdat. Jos
  muistiota ei ole eikä laajemmalle kontekstille ole tarvetta, CLAUDE.md
  riittää.

Nyrkkisääntö: jos oppi muuttaa sitä MITEN tuleva sessio tekee työtä
(komento, kikka, ansa) → CLAUDE.md. Jos se kertoo MITÄ projektissa on
päätetty tai tehty → muistio.

## Vaiheet

### 1. Kerää ehdokkaat keskustelusta

Käy koko sessio läpi ja poimi ehdokasopit. Etsi erityisesti:

- **Ympäristöansat**: työkalu epäonnistui ensin ja kiertotie löytyi
  kokeilemalla (esim. polku, lippu, versioero, API-raja). Nämä ovat
  arvokkaimpia — sama kokeilu toistuisi muuten joka sessiossa.
- **Testauksen sudenkuopat**: mikä näytti virheeltä muttei ollut (tai
  päinvastoin), ja miten asia todennetaan oikein.
- **Sessiossa syntyneet konventiot**: sanamuoto-, tyyli- tai
  arkkitehtuuripäätökset jotka sitovat jatkoa.
- **Prosessioivallukset**: nopeampi tapa tehdä toistuva asia.

Suodata armottomasti — ehdota vain KORKEAN ARVON oppeja. Hylkää itse:
kertaluontoiset yksityiskohdat, asiat jotka on jo muistissa, ja kaikki minkä
tuleva sessio päättelee koodista sekunneissa. Vähemmän ja parempia on
parempi kuin kattava lista: jokainen turha muistirivi on pysyvä kulu.

### 2. Lue nykyinen muisti ennen ehdottamista

Lue `CLAUDE.md` (jos on) ja silmäile muistion tuoreimmat osiot, jotta et
ehdota duplikaatteja. Merkitse samalla **auditointiehdokkaat**: vanhentuneet
rivit, kolmeksi eri opiksi hajonnut sama asia (ehdota yhdistämistä),
turhan pitkät selitykset (ehdota tiivistämistä).

### 3. Ehdota hyväksyttäväksi/hylättäväksi

Esitä opit AskUserQuestion-työkalulla, multiSelect päällä, niputettuna
(max 4 vaihtoehtoa per kysymys; useampi kysymys jos oppeja on enemmän).
Jokaisessa vaihtoehdossa:

- **label**: lyhyt otsikko (esim. "Playwright: executablePath")
- **description**: TÄSMÄLLEEN se 1–2 rivin teksti joka tallennettaisiin,
  ja kohde (CLAUDE.md vai muistio). Käyttäjä hyväksyy tekstin, ei ideaa —
  ei yllätyksiä kirjoitusvaiheessa.

Auditointiehdotukset (poistot/yhdistämiset) samaan tapaan omana
kysymyksenään, jos niitä on. Jos ei ole, älä ehdota mitään seremoniallisesti.

### 4. Kirjoita hyväksytyt

- CLAUDE.md: lisää/luo `## Opit`-osio, yksi luettelomerkkirivi per oppi,
  tiiviisti. Jos tiedosto puuttuu, luo minimirunko: mikä projekti, keskeiset
  komennot, deploy/julkaisupolku jos on, Opit-osio.
- Muistio: lisää oikeaan osioon muistion omaa tyyliä noudattaen.
- Toteuta hyväksytyt auditointimuutokset (poistot/tiivistykset) samalla.

### 5. Commitoi ja raportoi

Jos projekti on git-repo ja muistitiedostot versioidaan, committaa
muutokset repon normaalin käytännön mukaan. Kerro lopuksi yhteenveto:
montako oppia tallennettiin, montako hylättiin, mitä auditoitiin.

## Esimerkki hyvästä CLAUDE.md-opista

```
- Playwright: käytä esiasennettua Chromiumia
  `executablePath:'/opt/pw-browsers/chromium-<versio>/chrome-linux/chrome'` —
  `npx playwright install` ei toimi (versioero), NODE_PATH=<repo>/node_modules
  scratchpad-skripteille.
```

Yksi rivi, täsmällinen polku, syy mukana — tuleva sessio ei polta yhtään
kierrosta saman selvittämiseen.
