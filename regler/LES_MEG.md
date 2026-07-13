# Koderegler — vedlikeholdsfil

Denne filen styrer hvilke koder som gjelder når, og hvilken provisjon hver kode
skal utløse. **Koder hardkodes aldri i scriptet** — alt ligger her.

## Eierskap og distribusjon

- **Én eier** av filen (én person redigerer).
- Distribueres sentralt via SharePoint/OneDrive. Andre butikker bruker en
  *lesekopi*.
- Versjoner endres ved å legge til nye rader med ny `gyldig_fra`, ikke ved å
  overskrive gamle. Da kan historiske salg avstemmes mot regelen som gjaldt på
  *salgsdatoen*.

## Kolonner

| Kolonne              | Format        | Forklaring |
|----------------------|---------------|------------|
| `kode`               | tekst         | Provisjonsutløsende kode i produksjonssystemet. |
| `beskrivelse`        | tekst         | Lesbar tekst (kun for mennesker). |
| `operator`           | tekst         | Telia / Telenor / … |
| `forventet_provisjon`| heltall       | Forventet verdi koden skal utløse. |
| `valuta`             | tekst         | NOK. |
| `gyldig_fra`         | `ÅÅÅÅ-MM-DD`  | Første dag regelen gjelder (inklusiv). |
| `gyldig_til`         | `ÅÅÅÅ-MM-DD`  | Siste dag regelen gjelder (inklusiv). **Tom = gjelder fremdeles.** |

## Regler for redigering

1. Skal en sats endres fra en dato: sett `gyldig_til` på den gamle raden til
   dagen før, og legg til en **ny rad** med ny `gyldig_fra` og ny sats.
   (Se `ABO5GS` i `koderegler.csv` som eksempel.)
2. Overlappende perioder for samme `kode` er en feil — scriptet advarer.
3. Filen er semikolon-separert (`;`) fordi norske tall/Excel ofte bruker komma.

> Foretrekker du Excel framfor CSV: lagre som `koderegler.xlsx` med samme
> kolonner. `src/regler.py` leser begge.

---

# Kode-register (`koder_tele.csv`)

Den **autoritative lista over alle gyldige telekoder**, importert fra den sentrale
Excel-kildefilen (arkene Telia/ice/Telenor). Dette er fasiten for hvilke koder som
finnes — brukes til å fange «ukjent kode» og som kilde for mapping/provisjon.

**Genereres automatisk** — rediger ikke for hånd. Kjør på nytt når kode-arket
oppdateres sentralt:

```
python -m verktoy.importer_koder "/sti/til/Book1.xlsx"
```

## Kolonner
| Kolonne      | Forklaring |
|--------------|------------|
| `kode`       | Provisjonskoden (som registreres i Blueberry). |
| `operator`   | Telia / ice / Telenor. |
| `kategori`   | Grovklasse (utledet): abonnement / tilbehorsbinding / delbetaling / mbb / tilleggstjeneste. Omtrentlig — `betegnelse`/`merknad` er fasit. |
| `betegnelse` | Tekst fra kilden (f.eks. «TELIA X START med avtaletid»). |
| `imei`       | Telia: JA = IMEI registreres (terminalbinding m/enhet). |
| `abo_type`   | Telia/Telenor A/B/C (provisjonsklasse i kilden). |
| `gyldig_fra` / `gyldig_til` | **Rå datoer fra kilden** — blandet format (`2.06`, `1.9.25`, `2023-03-17`, `25/03/2026`). Må ryddes før de brukes til datostyrt oppslag. |
| `merknad`    | Kommentar fra kilden (binding/karantene/«ingen provisjon ved utstyrsendring» osv.). |

> **Ikke i kilden:** provisjons*beløp* er nesten ikke fylt inn i arket, så
> `forventet_provisjon` (i `koderegler.csv`) kan ikke fylles herfra ennå.

---

# Abonnement→kode-mapping (`abonnement_kode.csv`)

Operatørportalene viser abonnementet som fri tekst («iceMax Kampanje 12 mnd»),
mens Blueberry registrerer en **kode**. Denne filen oversetter tekst → kode.
Leses av `src/kodemapping.py`. Samme eier-/distribusjonsregler som over.

## Kolonner

| Kolonne       | Forklaring |
|---------------|------------|
| `operator`    | ice / Telia / Telenor. |
| `monster`     | Delstreng som må finnes i abonnementsteksten fra portalen (små bokstaver, kollapset mellomrom). F.eks. `telia x start`. |
| `salgsform`   | `uten` (uten binding), `terminal` (telefon m/IMEI), `tilbehor` (tilbehørsbinding), `binding` (ice: 12-mnd-variant), `any`. |
| `kundetype`   | `ny` (nytegning/portering), `eks` (oppgradering), `any`. Brukes for Telia tilbehørsbinding (NY/EKS). |
| `kode`        | Blueberry-koden. Må finnes i `koder_tele.csv`. |
| `gyldig_fra` / `gyldig_til` | Salgsdato-periode (kan stå tom). |
| `merknad`     | Notat for mennesker. |

## Slik virker oppslaget (`src/kodemapping.py`)

1. **Salgsform avledes** fra portalfeltene: ingen binding → `uten`; ice + binding →
   `binding`; Telia + enhet/IMEI → `terminal`; Telia + binding uten enhet → `tilbehor`.
2. Rad matcher når `operator`, `monster` (delstreng) og `salgsform` stemmer. Mest
   spesifikke `monster` vinner (lengste treff) — `telia x max pluss` slår `telia x max`.
3. **ice**: binding er IKKE egen kode — `binding` velger `…12`-varianten. Unntak:
   «iceMax Kampanje» har ikke «binding» i teksten ⇒ `uten` ⇒ `SICEMAX` (rabatt).
4. **Telia terminalbinding**: abonnementskoden (`…AVT`) + **IMEI-kode** (`SNC400`/
   `SNC200`) slås opp pr. modell i `imei_modell.csv` og legges til som egen kode.
5. **Telia tilbehørsbinding**: NY vs EKS. Er kundetype ukjent, antas `ny`
   (`kundetype_antatt=True`).
6. Treffer ingen rad, flagges `uavklart` i stedet for å forsvinne.

## `imei_modell.csv`
Genereres av importøren (IMEI-arket). Modell → Telia IMEI-kode. Modeller som ikke
står der, har ingen IMEI-kode (kun abonnementskoden registreres).

---

# Ekstrakode-utløsere (`ekstrakoder.csv`)

Elkjøp-interne tilleggskoder (SPK26…) som utløses av en abonnementskode. Leses av
`src/kodemapping.py` (`les_ekstrakoder`) — endringer her er live overalt ved neste
oppstart, uten ny zip.

| Kolonne      | Forklaring |
|--------------|------------|
| `ekstrakode` | Tilleggskoden som forventes i Blueberry (f.eks. SPK26TEXTILB). |
| `operator`   | ice / telia / telenor. |
| `utloser`    | Abonnementskode-mønster, `*` = wildcard (f.eks. `TEX*TILNY` = alle Telia X tilbehørsbinding NY). |
| `gyldig_fra` / `gyldig_til` | Kampanjeperiode — koden dør av seg selv etter `gyldig_til`. Tom = alltid. |
| `merknad`    | Notat for mennesker. |

> Familie-kodene (SPK26ICEFAM/SPK26TEFAM = antalls-/ordrebaserte) og TelePeak-
> betingelsene (SPK26TNOPP = oppsalg-signal) er logikk og ligger fortsatt i koden.

> **Dekning:** Telia-radene er komplette (alle plan × salgsform). **ice** har foreløpig
> bare de planene vi har sett portaltekst for (iceMax, Mobil 5/18 GB) — legg til
> resten når portalteksten er bekreftet. **Telenor** mangler (egen portal + forsikring
> må kartlegges). Kampanjekoder (TEXSTART12*) er ikke lagt inn (usikker portaltekst).
