# Aanmelden op Fedora met je Belgische eID via GDM — een werkende handleiding

Deze handleiding beschrijft een **werkende** opzet om grafisch in te loggen op Fedora
(via GDM, het GNOME-inlogscherm) met je Belgische elektronische identiteitskaart, in
plaats van met je wachtwoord. Je steekt je eID in de kaartlezer, GDM vraagt je pincode,
en je wordt aangemeld.

De aanpak gebruikt **SSSD** (de moderne, ondersteunde manier op Fedora) met de
**OpenSC**-PKCS#11-module, en de PAM-stack wordt beheerd met **authselect**.

Deze handleiding bevat ook een sectie "Valkuilen" met de problemen die je vrijwel zeker
tegenkomt, want verschillende stappen werken niet op de naïeve manier. Lees die sectie;
ze bespaart je uren.

> **Belangrijke veiligheidsmaatregel.** Houd tijdens het hele proces een tweede
> terminal open waarin je root bent (`sudo -i`), en gebruik de `with-smartcard`-
> instelling (niet `with-smartcard-required`) zodat je wachtwoord als terugval blijft
> werken. Zo sluit je jezelf niet buiten.

---

## Waarom niet `pam_pkcs11`?

Oudere handleidingen verwijzen naar `pam_pkcs11`. Dat pakket levert recente Fedora niet
meer; smartcard-authenticatie loopt nu via SSSD (`pam_sss`). Die aanpak is bovendien
veiliger. Deze handleiding gebruikt dus uitsluitend SSSD.

---

## Waarom OpenSC en niet de officiële eID-middleware (`beid`)?

Tegenintuïtief, maar belangrijk: voor de **systeemlogin** werkt de **OpenSC**-module
(`opensc-pkcs11.so`) betrouwbaar, en de officiële eID-middleware (`beidpkcs11.so`)
**niet**. SSSD voert de kaartlezing uit in een afgeschermd, onbevoorrecht hulpproces
(`p11_child`, draaiend als de `sssd`-systeemgebruiker). De `beid`-middleware functioneert
daar niet correct — ze rapporteert "token not present" terwijl de kaart in de lezer zit.
OpenSC is wél ontworpen om in zo'n context te werken en is op Fedora de geteste module.

Je kunt de eid-mw/`beid`-middleware gerust geïnstalleerd houden voor de eID-viewer en
voor ondertekenen in de browser — die toepassingen draaien in je gewone gebruikerssessie,
waar `beid` wél werkt. Voor de login gebruik je OpenSC.

---

## 1. Pakketten installeren

```bash
sudo dnf install sssd sssd-tools opensc pcsc-lite pcsc-tools authselect
```

Activeer de smartcard-daemon:

```bash
sudo systemctl enable --now pcscd
```

Steek je eID in en controleer dat OpenSC de kaart herkent:

```bash
opensc-tool --name        # moet "Belpic cards" tonen
opensc-tool --list-readers
```

---

## 2. Eén PKCS#11-module: schakel de andere uit

Als zowel OpenSC als de eid-mw-middleware geïnstalleerd zijn, registreren ze **beide**
een PKCS#11-module bij p11-kit. SSSD laadt ze dan allebei, en ze vechten om de ene
kaartlezer. Het gevolg is `device error` / `token not present`-fouten.

Houd daarom enkel OpenSC actief. Schakel de `beid`-module uit de p11-kit-proxy met een
override in `/etc/pkcs11/modules/`. Zoek eerst de bestandsnaam:

```bash
grep -rl beid /usr/share/p11-kit/modules/
```

Dat geeft meestal `/usr/share/p11-kit/modules/beid.module`. Maak een override met
**exact dezelfde bestandsnaam** in `/etc`, met een lege `enable-in:` (dat schakelt de
module volledig uit):

```bash
sudo mkdir -p /etc/pkcs11/modules
sudo tee /etc/pkcs11/modules/beid.module >/dev/null <<'EOF'
module: /usr/lib64/pkcs11/beidpkcs11.so
enable-in:
EOF
```

> Heb je de eid-mw-middleware niet geïnstalleerd, dan kun je deze stap overslaan.

---

## 3. OpenSC: maak het eID-certificaat leesbaar zonder PIN

Dit is een cruciale, niet voor de hand liggende stap.

SSSD leest het certificaat van de kaart in een **pre-auth**-fase — dat wil zeggen
*vóór* je je pincode ingeeft. Maar OpenSC behandelt het eID-certificaat standaard als
"privé" (PIN vereist), waardoor het in die fase onzichtbaar is en de login mislukt.

Je dwingt OpenSC het certificaat als publiek leesbaar te presenteren met de optie
`private_certificate = declassify` in `/etc/opensc.conf`.

> Let op: de waarde moet **`declassify`** zijn. De waarde `ignore` doet het
> tegenovergestelde (ze verbergt het certificaat).

Bewerk `/etc/opensc.conf`:

```bash
sudo nano /etc/opensc.conf
```

Zorg voor een `app default`-blok met:

```
app default {
    framework pkcs15 {
        private_certificate = declassify;
    }
}
```

Controleer dat het certificaat nu **zonder** PIN zichtbaar is (er mag geen pincode
gevraagd worden):

```bash
pkcs11-tool --module /usr/lib64/pkcs11/opensc-pkcs11.so --list-objects --type cert
```

Je moet nu het `Authentication`-certificaat zien verschijnen zonder loginprompt.

---

## 4. CA-certificaten plaatsen

SSSD valideert standaard de certificaatketen. Plaats daarvoor de Belgische
overheids-CA's (Root CA + Citizen CA, in PEM) in de standaard-CA-database van SSSD.
Haal ze van **https://repository.eid.belgium.be**.

```bash
sudo mkdir -p /etc/sssd/pki
sudo sh -c 'cat belgiumrca*.pem citizenca*.pem > /etc/sssd/pki/sssd_auth_ca_db.pem'
```

(Vervang de bestandsnamen door wat je gedownload hebt.) De eID gebruikt geen voor jou
bereikbare OCSP bij lokale login; daarom zetten we OCSP-controle uit in de SSSD-config
(volgende stap).

---

## 5. Het rijksregisternummer uit je certificaat halen

Je koppelt de kaart aan je account met een certmap-regel die op je
**rijksregisternummer** matcht (het `serialNumber`-veld in het certificaat).

Lees je authenticatiecertificaat uit (met je kaart in de lezer):

```bash
pkcs11-tool --module /usr/lib64/pkcs11/opensc-pkcs11.so \
  --read-object --type cert --label "Authentication" --output-file authcert.der
openssl x509 -inform DER -in authcert.der -noout -subject
```

In de `subject`-regel staat `serialNumber=` gevolgd door je rijksregisternummer. Noteer
dat nummer.

---

## 6. SSSD configureren

Maak `/etc/sssd/sssd.conf`:

```bash
sudo nano /etc/sssd/sssd.conf
```

Inhoud (vervang `jc` door je gebruikersnaam en `<RIJKSREGISTERNUMMER>` door je nummer):

```ini
[sssd]
services = nss, pam
domains = local
certificate_verification = no_ocsp

[nss]

[pam]
pam_cert_auth = True

[domain/local]
id_provider = proxy
proxy_lib_name = files
proxy_pam_target = sssd-shadowutils
proxy_fast_alias = True
local_auth_policy = only

[certmap/local/jc]
matchrule = <SUBJECT>.*serialNumber=<RIJKSREGISTERNUMMER>.*
```

**Twee regels die het verschil maken, en die makkelijk over het hoofd gezien worden:**

- `id_provider = proxy` met `proxy_lib_name = files`: op recente Fedora bestaat
  `id_provider = files` niet meer; je krijgt dan een ongeldig-domein-fout en SSSD start
  niet. De `proxy`-provider laat SSSD je lokale `/etc/passwd`-account gebruiken.
- `local_auth_policy = only`: **dit is de sleutel.** Zonder deze regel valt SSSD na de
  geslaagde smartcard-fase terug op een wachtwoordcontrole (`pam_unix`), waarbij je
  ingetoetste pincode als wachtwoord wordt gecontroleerd — wat mislukt. `only` zegt dat
  enkel de lokale (smartcard-)authenticatie geldt, zonder die wachtwoord-terugval.

Zet de juiste permissies en start SSSD:

```bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
sudo systemctl enable --now sssd
sudo systemctl restart sssd
sudo sssctl config-check     # moet schoon zijn
```

> Controleer dat het PAM-doel bestaat dat de proxy gebruikt:
> `ls -l /etc/pam.d/sssd-shadowutils` (hoort meegeleverd te zijn met `sssd`).

---

## 7. De koppeling kaart ↔ gebruiker testen (vóór je PAM aanraakt)

Controleer dat je certificaat naar je gebruiker mapt. Dit raakt de kaart niet aan en
sluit je nergens buiten:

```bash
# kale base64 van het certificaat (zonder BEGIN/END-regels):
openssl x509 -inform DER -in authcert.der -out authcert.pem
CERT=$(grep -v -- '-----' authcert.pem | tr -d '\n')
sudo sssctl cert-map "$CERT"
```

Dit moet je **gebruikersnaam** teruggeven. Zo niet, controleer dan je `matchrule` en je
rijksregisternummer in stap 6.

> Opmerking: `sssctl user-checks ... -s gdm-smartcard` lijkt handig als test, maar dat
> commando handelt de pincode-conversatie niet altijd goed af en kan "Please (re)insert
> Smartcard" tonen terwijl de echte GDM-login wél werkt. Vertrouw voor de eindtest op de
> echte GDM-login, niet op `user-checks`.

---

## 8. Smartcard-login inschakelen met authselect

Fedora beheert de PAM-stacks met `authselect`. De smartcard-functie zit in het
**`sssd`-profiel** (niet in `local`). Schakel over naar dat profiel mét de
smartcard-feature:

```bash
sudo authselect select sssd with-smartcard --backup=voor-eid
```

Als authselect klaagt over bestaande aanpassingen, voeg `--force` toe.

Controleer:

```bash
authselect current        # Profile ID: sssd, met with-smartcard
```

Bevestig dat je account nog gevonden wordt vóór je uitlogt:

```bash
id
getent passwd jc
```

---

## 9. Testen

1. Open een **root-terminal** als vangnet (`sudo -i`) en laat die open staan.
2. Steek je eID in de lezer.
3. Log uit (of "Gebruiker wisselen") zodat je het GDM-inlogscherm krijgt.
4. GDM vraagt nu je **pincode** in plaats van je wachtwoord. Geef de pincode.
5. Je wordt aangemeld.

Omdat je `with-smartcard` (niet `-required`) gebruikt, kun je altijd nog je gewone
wachtwoord intikken als de kaart-login eens faalt.

---

## 10. Optioneel: wachtwoord uitschakelen (kaart verplicht)

Werkt de eID-login een paar dagen betrouwbaar, en wil je dat aanmelden enkel met de
kaart kan:

```bash
sudo authselect enable-feature with-smartcard-required
sudo authselect apply-changes
```

Terugdraaien:

```bash
sudo authselect disable-feature with-smartcard-required
sudo authselect apply-changes
```

---

## Valkuilen (samengevat)

Dit zijn de problemen die je vrijwel zeker tegenkomt, met hun oplossing:

1. **`pam_pkcs11` bestaat niet meer** → gebruik SSSD.
2. **Twee PKCS#11-modules vechten om de lezer** (`device error 48` / `token not present`)
   → schakel `beid` uit de p11-kit-proxy, houd enkel OpenSC (stap 2).
3. **OpenSC verbergt het certificaat achter de PIN** (pre-auth ziet niets) → zet
   `private_certificate = declassify` in `/etc/opensc.conf` (stap 3). Let op: `ignore`
   is fout, `declassify` is juist.
4. **`id_provider = files` is ongeldig** op recente Fedora (SSSD start niet) → gebruik
   `id_provider = proxy` met `proxy_lib_name = files` (stap 6).
5. **De `with-smartcard`-feature bestaat niet in het `local`-profiel** → schakel over
   naar het `sssd`-profiel (stap 8).
6. **Na een geslaagde PIN volgt toch een wachtwoordcontrole die mislukt**
   (`pam_unix ... password check failed`) → voeg `local_auth_policy = only` toe (stap 6).
   Dit was vaak de allerlaatste ontbrekende schakel.
7. **`sssctl user-checks` faalt terwijl GDM werkt** → vertrouw op de echte GDM-login als
   eindtest.

---

## Veiligheidsopmerkingen

- **`declassify`** maakt je authenticatiecertificaat zonder PIN leesbaar voor wie fysiek
  toegang heeft tot de kaart in de lezer. Dat is voor een inlogcertificaat normaal geen
  bezwaar: de bijhorende **privésleutel** blijft door de pincode beschermd en is zonder
  die code onbruikbaar.
- Gebruik tijdens het opzetten altijd `with-smartcard` (wachtwoord als terugval), niet
  `with-smartcard-required`.
- Zet `no_verification` **nooit** permanent in `sssd.conf`. Als je dat tijdens het
  debuggen gebruikt hebt, verwijder het en laat enkel `no_ocsp` staan, met een correcte
  CA-bundel (stap 4).
