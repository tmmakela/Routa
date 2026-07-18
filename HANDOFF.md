# ROUTA — kehityksen jatkuvuusdokumentti

> **Lue tämä ensin joka uudessa sessiossa.** Tämä dokumentti kertoo mikä
> ROUTA on, missä mennään nyt, ja miten työtä jatketaan siitä mihin
> viimeksi jäätiin. Päivitä **Sessioloki** (alhaalla) aina kun teet
> muutoksia, niin seuraava kerta pystyy jatkamaan saumattomasti.

**Nykyversio:** v0.7.5
**Repo:** `tmmakela/Routa` · **Päähaara:** `main` · **Kehityshaara:** `claude/syntikka-projekti-yn2uhl`
**Koko projekti on yhdessä tiedostossa:** [`index.html`](./index.html)

---

## 1. Mikä ROUTA on

ROUTA on selaimessa toimiva "talvisyntikka" (Jääportit-henkinen). Se on
tehty **puhtaalla Web Audiolla ilman kirjastoja**, kaikki yhdessä
HTML-tiedostossa. Estetiikka: pohjoinen / talvinen hardware-paneeli
(tummansininen, jääsininen ja revontuli-vihreä, fontit Fjalla One +
IBM Plex Mono).

Ominaisuudet lyhyesti:
- 8-ääninen polyfonia (osci 1 + osci 2 + kohina per ääni)
- Per-ääni lowpass-filtteri omalla verhokäyrällä
- FX-ketju: drive → chorus → delay → reverb → EQ → kompressori
- LFO (filtteri tai pitch), 16-askeleinen sekvensseri sävellajilukolla
- Generatiivinen **KAAMOS**-moodi sekvensserissä
- Näppäimistö (hiiri/kosketus glissandolla, QWERTY, Web MIDI)
- Kategorisoitu **preset-kirjasto** (LEADS / BASS / PAD / KEYS)
- Presettien JSON export/import

---

## 2. Miten ajat ja testaat

### Ajaminen
Avaa `index.html` **Chromessa** (paras tuki; Web MIDI ei toimi Safarissa).
Paina **POWER** tai kosketa mitä tahansa → AudioContext käynnistyy
(vaaditaan käyttäjän ele iOS:n takia). Soita Z- ja Q-riveillä tai
koskettimella.

### Automaattinen tarkistus (headless-selain)
Ympäristössä on Chromium + Playwright valmiina. Näin tarkistat ettei
mikään mene rikki (ei JS-virheitä, presetit latautuvat):

```js
// aja: node check.cjs
const { chromium } = require('/opt/node22/lib/node_modules/playwright');
(async () => {
  const b = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium' });
  const p = await b.newPage({ viewport:{width:1080,height:940} });
  const errs=[];
  p.on('pageerror', e => errs.push('PAGEERROR: '+e.message));
  await p.goto('file:///home/user/Routa/index.html');
  await p.waitForTimeout(400);
  // klikkaa jokainen preset, varmista ettei heitä virhettä
  const ids = await p.$$eval('#presetLib .pchip', els => els.map(e=>e.dataset.preset));
  for (const id of ids) { await p.click(`#presetLib .pchip[data-preset="${id}"]`); await p.waitForTimeout(20); }
  await p.screenshot({ path:'shot.png' });   // silmämääräiseen tarkistukseen
  console.log('PRESETS:', ids.length, '| ERRORS:', errs.length?errs.join('\n'):'none');
  await b.close();
})();
```

> Huom: konsolissa näkyvä `ERR_CONNECTION_RESET` / "Failed to load resource"
> on vain Google Fonts -latauksen esto sandboxissa — **ei** koodivirhe.
> Suodata ne pois kuten aiemmissa tarkistuksissa.

**Tee tämä tarkistus + kuvakaappaus aina ennen committia** kun muutat UI:ta
tai preset-dataa.

---

## 3. Arkkitehtuuri

Yksityiskohtaisin, aina ajan tasalla oleva kuvaus on `index.html`:n alussa
olevassa `HANDOFF`-kommentissa (rivit ~7–102) sekä versiokohtaisessa
changelogissa. Tässä tiivistys signaalireitistä:

```
Voice(osc1 + osc2 + noise)
  -> voiceGain (ADSR: atk/dec/sus/rel)
  -> vf: per-ääni lowpass + AD-verhokäyrä (detune-centteinä)
  -> bus
     -> DRIVE (WaveShaper, curve (1+k)x/(1+k|x|), k=drive*60, 2x -> tone LP)
     -> CHORUS (2 moduloitua delaylinjaa L/R + feedback)
     -> dry + delaySend + reverbSend
     -> EQ (lowshelf 250 Hz / highshelf 4 kHz)
     -> DynamicsCompressor (glue: -18 dB, 3:1, knee 12)
  -> master -> analyser -> destination
```

- **LFO:** yksi oskillaattori, kohde `filter` tai `pitch`, kytketty per ääni.
- **Polyfonia:** max 8 (`MAXV`), vanhin ääni varastetaan.
- **Sekvensseri:** 16 askelta, lookahead-scheduler (setInterval 30 ms,
  120 ms ikkuna). KAAMOS = generatiivinen satunnaiskävely sävellajissa.
- **Ei localStoragea** (ei toimi Claude-artifacteissa) → presetit ovat
  muistissa + JSON-tiedostoina.

### Parametrit (objekti `P`)
Kaikki äänen parametrit ovat yhdessä objektissa `P` (oletukset
`DEFAULTS`). Preset = `P`:n kopio jossa osa arvoista ylikirjoitettu.

| Ryhmä | Kentät |
|-------|--------|
| Osc 1 | `osc1wave, osc1oct, osc1lvl` |
| Osc 2 | `osc2wave, osc2oct, osc2det` (centtiä), `osc2lvl` |
| Kohina | `noise` |
| Voice | `glide` (s), `unison` (1..7), `uniDetune` (spread, centtiä) |
| Filtteri | `cutoff, reso`, verhokäyrä `fAmt, fAtk, fDec` |
| LFO | `lfoRate, lfoDepth, lfoTarget` (`'filter'`/`'pitch'`) |
| Amp ADSR | `atk, dec, sus, rel` |
| Delay | `dlyTime, dlyFb, dlyMix` |
| Reverb | `rvSize, rvDecay, rvMix` |
| Drive | `drive, driveTone` |
| Chorus | `chRate, chDepth, chFb, chMix` |
| EQ / Master | `eqLow, eqHigh, master` |

> `lfoTarget:'pitch'` → efektiivinen vibrato = `lfoDepth * 0.05` senttiä.
> Esim. `lfoDepth:120` ≈ 6 senttiä. LFO on aina päällä (ei onset-viivettä).

---

## 4. Preset-järjestelmä (miten lisätään uusia)

Kaksi rakennetta, molemmat `index.html`:n `<script>`-lohkossa:

1. **`PRESETS`** — litteä `id → parametrit` -objekti. Jokainen preset
   levittää oletukset ja ylikirjoittaa: `{...P, cutoff:3200, ...}`.
2. **`LIBRARY`** — kategorioiden metadata, määrää mitä ja missä
   järjestyksessä palkissa näkyy:
   ```js
   { cat:'LEADS', items:[ {id:'tulikettu', name:'Tulikettu'}, ... ] }
   ```

`buildPresetBar()` renderöi palkin `LIBRARY`:sta. Kategorian accent-väri
tulee `CAT_ACCENT`-mäpistä (→ `:root`-muuttuja `--acc-lead/bass/pad/keys`).

### Uuden presetin lisäys
1. Lisää parametrit `PRESETS`-objektiin uudella id:llä.
2. Listaa se `LIBRARY`-taulukossa haluttuun kategoriaan (`{id, name}`).
3. Nappi tulee automaattisesti. Aja tarkistus + kuvakaappaus.

### Uuden kategorian lisäys
1. Määrittele accent-väri `:root`:iin (esim. `--acc-fx:#....`).
2. Lisää se `CAT_ACCENT`-mäppiin (`FX:'var(--acc-fx)'`).
3. Lisää uusi ryhmä `LIBRARY`-taulukkoon.

### Nimeämiskäytäntö
Presetit ovat suomalaisia/pohjoisia teemanimiä (Halla, Ahma, Tunturi,
Revontuli, Tulikettu…). Pidä id pienellä ja ilman ääkkösiä
(`tera`, `id` ≠ näyttönimi `Terä`).

---

## 5. Nykytila — mitä on tehty

- ✅ Ydinsyntikka + FX-ketju + sekvensseri + näppäimistö + MIDI (≤ v0.5.3)
- ✅ **Preset-kirjasto kategorisoituna** (v0.6.0): LEADS / BASS / PAD / KEYS
- ✅ **Vahva LEADS-setti** (9): Tulikettu, Terä, Sisu, Kaiku, Loitsu,
  Hehku, Ukko, Viima, Pakkanen
- ✅ **Kirjaston visuaalinen uudistus** (v0.6.1): kehystetty paneeli,
  värikoodatut kategoriat, hehkuvat chipit, aktiivisen presetin korostus
- ✅ **iPad/iOS-kosketuksen kovennus** (v0.6.2)
- ✅ **Unison + glide -moottori** (v0.7.0): VOICE-moduuli (glide/unison/spread),
  supersaw-äänet. Kirjasto 23 presettiä (LEADS 10, BASS 3, PAD 6, KEYS 4).
- ⚠️ BASS (3) & KEYS (4) vielä kevyempiä kuin LEADS — voisi täydentää

---

## 6. Roadmap / NEXT (mistä jatkaa)

Priorisoitu; poimi ylhäältä. (`index.html`:n "NEXT (ideas)" -lista on sama.)

1. **Täydennä kirjastoa** — BASS (3) ja KEYS (4) LEADSin (10) tasolle.
   Ehkä uudet kategoriat: PLUCK, FX.
2. ~~Glide / portamento~~ — ✅ tehty v0.7.0 (`glide`-param, `exponentialRampToValueAtTime`).
3. ~~Unison-detune~~ — ✅ tehty v0.7.0 (`unison` + `uniDetune`, supersaw).
4. ~~Sekvenssi presettiin~~ — ✅ tehty v0.7.3 (export/import niputtaa seq:n).
5. **Preset-kategoria JSONiin** — export/import muistaisi kategorian.
6. ~~WAV-tallennus~~ — ✅ tehty v0.7.2 (ScriptProcessor-tap + encodeWAV).
7. **AudioWorklet drive/foldback** — rosoisempaa säröä (ja WAV-tap workletiksi).
8. **Easy-tila / ohjaava ensikäyttö** — makro-nupit ja aloitusopastus.

---

## 7. Työskentelytapa (git & versiointi)

- **Kehitä** haarassa `claude/syntikka-projekti-yn2uhl`; käyttäjän toive on
  että muutokset viedään **suoraan `main`iin** (ei erillistä uploadia).
  Käytäntö tähän asti: commit → push `main` → `git branch -f` kehityshaara
  mainiin → push kehityshaara (pidetään synkassa).
- **Versiointi:** nosta versionumero neljässä paikassa:
  `index.html`-kommentin otsikko (rivi ~9), changelog-lohko,
  `<div class="badge">`, ja footer. Semverin henki: uusi feature = minor
  (0.6→0.7), hienosäätö/korjaus = patch (0.6.1→0.6.2).
- **Changelog:** lisää versiolohko `index.html`:n kommenttiin
  (arkkitehtoniset yksityiskohdat) JA rivi tämän dokumentin Sessiolokiin.
- **Testaa** aina headless-tarkistuksella + kuvakaappauksella ennen pushia.
- Älä laita mallin id:tä / sisäisiä yksityiskohtia committeihin tai koodiin.

---

## 8. Sessioloki

> Uusin ylimmäs. Merkitse: päivä, versio, mitä tehtiin, mihin jäätiin.

### 2026-07-18 (jatko 7) · v0.7.5 — Kompakti preset-kirjasto (välilehdet)
- 4 pinottua kategoriariviä → yksi välilehtipalkki. Otsikkorivi: PRESETS +
  kategoriavälilehdet (`.ptab`, count + accent-väri) + ikoni-export/import;
  vain aktiivisen kategorian chipit näkyvät alla.
- Paneelin korkeus ~185→77px (desktop), ~230→92px (iPad).
- JS: `activeCat` + `curPreset` tila; `buildPresetBar()` piirtää välilehdet,
  `renderChips()` aktiivisen kategorian, ja ladattu preset pysyy korostettuna
  välilehteä vaihdettaessa.
- Testattu: 4 välilehteä, kategorian vaihto näyttää oikeat chipit, lataus +
  korostus säilyy, korkeus tippui. 0 virhettä.
- **Seuraavaksi:** Easy-tila / ohjaava ensikäyttö, tai AudioWorklet-särö.

### 2026-07-18 (jatko 6) · v0.7.4 — iPad/tablet-optimointi
- Uusi `@media (pointer:coarse) and (min-width:601px)` -lohko (tabletit; puhelimet
  jäävät max-width:600-layouttiin). Sijoitettu peruscoarse-lohkon JÄLKEEN → voittaa.
- **Sticky-koskettimisto** viewportin pohjaan → voit soittaa säätäessäsi ylhäällä.
  Footer piilotettu tässä layoutissa jotta koskettimisto on tasan pohjassa
  (oli 26px irti footerin takia).
- **Suuremmat kosketuskohteet:** nupit 44px, preset-chipit, io/wave/seq-napit,
  liukusäätimet, root/scale-selectit, step-selectit, gatet 30px, octave-napit.
- Landscape (`orientation:landscape`): lyhyempi 165px koskettimisto → enemmän
  tilaa moduuleille.
- Testattu portrait (820×1180) + landscape (1180×820): koskettimisto pinnattu
  pohjaan, ei vaakaylivuotoa, kosketuskohteet suuremmat, soitto toimii. 0 virhettä.
- **Seuraavaksi:** Easy-tila / ohjaava ensikäyttö, tai AudioWorklet-särö.

### 2026-07-18 (jatko 5) · v0.7.3 — Koko projektin tallennus (params + sekvenssi)
- Export/import niputtaa nyt myös sekvensserin. Export: `{routa, params:{...P},
  seq:{bpm,gate,root,scale,kaamos,steps}}`. Import tunnistaa bundlen (`obj.params`)
  → `loadPreset(params)` + `applySeq(seq)`; muuten vanha litteä P (taaksepäin-
  yhteensopiva; preset-nappien litteät objektit toimivat yhä).
- `applySeq()` **mutatoi** `seq.steps`-objektit paikallaan (ei korvaa taulukkoa),
  jotta step-kahvojen closuret pysyvät ehjinä. `syncSeqUI()` päivittää
  bpm/gate/root/scale/kaamos + step-selectit ja gate-tilat.
- Testattu koko kierto (filechooser): export sisältää params+seq, import palauttaa
  bpm/root/scale/stepit/paramit, vanha litteä formaatti latautuu yhä. 0 virhettä.
- **Seuraavaksi:** Easy-tila / ohjaava ensikäyttö, tai AudioWorklet-särö.

### 2026-07-18 (jatko 4) · v0.7.2 — WAV-nauhoitus
- **REC-nappi** headerissa (POWERin vieressä): nauhoittaa master-ulostulon
  oikeaksi 16-bit stereo-WAV:ksi ja lataa sen (`routa-take.wav`).
- `boot()` lisää hiljaisen sivuhaaran: master → ScriptProcessor(4096, toimii
  iOS:llä) → 0-gain → destination. `onaudioprocess` kopioi raa'at Float32 L/R
  vain kun `recording=true`.
- `encodeWAV()` kirjoittaa RIFF/WAVE PCM-headerin + interleavatut 16-bit näytteet
  `ctx.sampleRate`:lla. `toggleRec()` vaihtaa tilan (REC/STOP) ja stopatessa
  rakentaa Blobin + lataa.
- Testattu oikealla download-kaappauksella: validi RIFF/WAVE, stereo, 44.1kHz,
  ei-hiljainen (huippunäyte ~22000). 0 virhettä.
- **Seuraavaksi:** Easy-tila / ohjaava ensikäyttö (helppokäyttö), sekvenssi
  presettiin, tai AudioWorklet-särö.

### 2026-07-18 (jatko 3) · v0.7.1 — Moderni ilme & tuntuma
- Käyttäjän toive: iisi, intuitiivinen, moderni → valittu suunta "moderni ilme + tuntuma".
- **Nuppien arvorenkaat:** jokaisen nupin ympärille SVG 270°-kaari joka täyttyy
  arvon mukaan (Vital/Serum-tyyli). `makeKnob` injektoi `<svg.ring>`, `render()`
  asettaa `rval` stroke-dasharray = norm*ARC.
- **Mikrovuorovaikutukset:** nuppi nousee+hehkuu vedettäessä (`.grab`), rengas
  kirkastuu hoverissa; moduulit korostuvat hoverissa; wave-napit nousevat;
  koskettimet hehkuvat painettaessa; POWER sykkii kunnes ääni käynnistetty
  (`prefers-reduced-motion` huomioitu).
- Testattu: 35 rengasta, täyttö seuraa arvoa, grab & hehku toimivat, 0 virhettä.
- **Seuraavaksi (jos jatketaan modernisointia):** Easy-tila (makro-nupit),
  ohjaava ensikäyttö, tai roadmapin toiminnot (sekvenssi presettiin, WAV).

### 2026-07-18 (jatko 2) · v0.7.0 — Unison + glide -moottori
- Iso äänellinen harppaus: **unison (supersaw)** ja **glide/portamento**.
- Uudet parametrit: `glide` (s), `unison` (1..7), `uniDetune` (spread centtiä).
- `noteOn` refaktoroitu: `mkOne()` luo yhden oscin, `mk()` rakentaa `unison`
  kopiota levitettynä ±`uniDetune`, taso 1/√unison. Voice tallentaa nyt
  `oscs`-taulukon (ennen o1/o2); `noteOff` ja pitch bend iteroivat sitä.
  `lastFreq` + `exponentialRampToValueAtTime` hoitaa glideñ. Oletukset
  unison:1/glide:0 → vanhat presetit soivat identtisinä.
- Uusi **VOICE-moduuli** (glide/unison/spread) OSC2:n ja FILTERin väliin.
- Kirjasto 15 → 23: +Kaiho (glide-lead), +Karhu/Jyrä (unison-basso),
  +Myrsky/Usva (supersaw/pehmeä pad), +Tuisku (unison-key); Ukko & Tulikettu
  päivitetty oikeiksi supersaw'iksi.
- Testattu: supersaw (unison 7 = 14 oscia/ääni) tuottaa signaalia, glide toimii,
  äänivaras kestää unisonin (9 nuottia), oletuspreset ennallaan. 0 virhettä.
- **Seuraavaksi:** täydennä BASS/KEYS, tai sekvenssi presettiin (Roadmap 1 & 4),
  tai WAV-tallennus (6).

### 2026-07-18 (jatko) · v0.6.2 — iPad/iOS-kosketuksen kovennus
- Korjattu ongelma jossa iPadilla pitkä painallus toi tekstin valinnan /
  iOS-callout-popupin / suurennuslasin soiton sijaan, ja tupla-tap zoomasi.
- `*`: `user-select:none` + `-webkit-touch-callout:none` koko sovellukseen.
- `button,select,input,label,a`: `touch-action:manipulation` (ei tupla-tap-zoomia);
  body jätetty `auto` → sivu vierittyy edelleen.
- `#kb *{touch-action:none}` → koskettimet (aiemmin vain `#kb`, joten `.key` oli
  `auto`) omaan gesture-hallintaan. Nupeilla oli jo `none`.
- Testattu iPad-emulaatiolla (kosketusnäyttö): computed-tyylit oikein, kosketus
  soittaa, ei virheitä. Huom: `-webkit-touch-callout` näkyy vain Safarissa, ei
  Chromiumin computed-tyyleissä — varmistettu lähdekoodista.

### 2026-07-18 · v0.5.4 → v0.6.1
- **v0.5.4:** 5 uutta presettiä (Halla, Ahma, Tunturi, Kide, Pakkanen).
- **v0.6.0:** Presetit kategorisoituun kirjastoon (`LIBRARY` + `CAT_ACCENT`
  + `buildPresetBar()`); lisätty koko LEADS-setti (8 uutta + Pakkanen).
- **v0.6.1:** Kirjaston visuaalinen uudistus — kehystetty `.preslib`-paneeli,
  otsikko + EXPORT/IMPORT, värikoodatut kategoria-accentit, `.pchip`-chipit
  reunavalolla ja aktiivisen hehkulla. Testattu desktop + mobiili, ei virheitä.
- **Luotu tämä `HANDOFF.md`.**
- **Seuraavaksi:** täydennä BASS/PAD/KEYS-setit (Roadmap kohta 1), tai
  aloita glide/portamento (kohta 2).

### (aiemmat, ennen jatkuvuusdokumenttia)
- v0.4.0–v0.5.3: UI-uudistus (rotary-knobit), FX-ketju (drive/chorus/eq/
  komp), iPad- ja puhelinlayoutit, EQ siirretty headeriin. Ks.
  `index.html`-changelog.
