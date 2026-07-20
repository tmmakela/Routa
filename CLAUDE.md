# ROUTA — projektiohjeet

ROUTA on yhden tiedoston selainsyntikka (`index.html`, puhdas Web Audio, ei
kirjastoja). Ei build-vaihetta: avaa `index.html` selaimessa (paras Chrome).
Kehityksen jatkuvuusdokumentti on **HANDOFF.md**.

## Opit

- **Lue HANDOFF.md ensin** — repon juuressa oleva lue-ensin
  jatkuvuusdokumentti: arkkitehtuuri, parametrit, roadmap ja sessioloki.
  Lue uuden session alussa; päivitä sessioloki + changelog lopuksi.
- **Playwright-testaus (käynnistys):** aja headless nodella
  `require('/opt/node22/lib/node_modules/playwright')` (EI `import`, ei
  paikallista pakettia); `chromium.launch({executablePath:'/opt/pw-browsers/chromium',
  args:['--autoplay-policy=no-user-gesture-required']})`. Ohita konsolivirhe
  `ERR_CONNECTION_RESET` / "Failed to load resource" (Google Fonts esto
  sandboxissa, ei oikea vika).
- **Testaus (ääni & iPad):** mittaa oikea äänisignaali patchaamalla
  `AudioNode.prototype.connect` (`addInitScript`) lisäämään AnalyserNode ennen
  destinationia → `getByteTimeDomainData`. iPad-emulaatio:
  `newContext({hasTouch:true,isMobile:true})` → `pointer:coarse`.
  `-webkit-touch-callout` ei näy Chromiumin computed-tyyleissä (Safari-only)
  → varmista lähteestä.
- **Git & versiointi:** muutokset suoraan `main`iin (ei manuaalista uploadia);
  commit → push main → `git branch -f claude/syntikka-projekti-yn2uhl main` →
  push kehityshaara (molemmat synkassa). Versio index.html:ssä 4 paikassa
  (yläkommentti, changelog, badge, footer) + HANDOFF.md "Nykyversio".
