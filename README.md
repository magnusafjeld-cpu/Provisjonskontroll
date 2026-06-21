# Elkjøp Provisjonskoder

Sentrale kode-/mapping-filer for **Provisjonskontroll**-verktøyet. Verktøyet henter
disse automatisk ved oppstart (offentlig raw-URL), så endringer her gjelder for
**alle salgsledere uten ny utrulling**.

Inneholder **ingen** persondata og **ingen** provisjonsbeløp — kun koder, mapping og
gyldighetsperioder.

## Filer (`regler/`)
- `koder_tele.csv` — registeret over alle gyldige telecom-koder.
- `abonnement_kode.csv` — portaltekst + salgsform → kode.
- `imei_modell.csv` — telefonmodell → Telia IMEI-kode (SNC400/SNC200).

## Oppdatere koder
Rediger fila her i GitHub (åpne fila → blyant-ikon → «Commit changes»). Neste gang
en salgsleder kjører verktøyet, hentes den nye versjonen automatisk.

## Format
Semikolon-separert (`;`), UTF-8. Kolonner og regler er dokumentert i hovedprosjektet
(`regler/LES_MEG.md`). Behold kolonneoverskriftene uendret.
