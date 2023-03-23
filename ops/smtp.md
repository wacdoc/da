# Byg din egen SMTP-mail-afsendelsesserver

## præambel

SMTP kan købe tjenester direkte fra cloud-leverandører, såsom:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali sky e-mail push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Du kan også bygge din egen mailserver - ubegrænset afsendelse, lave samlede omkostninger.

Nedenfor demonstrerer vi trin for trin, hvordan man bygger vores egen mailserver.

## Servervalg

Den selv-hostede SMTP-server kræver en offentlig IP med porte 25, 456 og 587 åbne.

Almindeligt brugte offentlige skyer har blokeret disse porte som standard, og det kan være muligt at åbne dem ved at udstede en arbejdsordre, men det er trods alt meget besværligt.

Jeg anbefaler at købe fra en vært, der har disse porte åbne og understøtter opsætning af omvendte domænenavne.

Her anbefaler jeg [Contabo](https://contabo.com) .

Contabo er en hostingudbyder baseret i München, Tyskland, grundlagt i 2003 med meget konkurrencedygtige priser.

Hvis du vælger Euro som købsvaluta, vil prisen være billigere (en server med 8 GB hukommelse og 4 CPU'er koster omkring 529 yuan om året, og det første installationsgebyr er gratis i et år).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Når du afgiver en ordre, bemærk `prefer AMD` , og serveren med AMD CPU vil have bedre ydeevne.

I det følgende vil jeg tage Contabos VPS som eksempel for at demonstrere, hvordan man bygger sin egen mailserver.

## Ubuntu systemkonfiguration

Operativsystemet her er Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Hvis serveren på ssh viser `Welcome to TinyCore 13!` (som vist i figuren nedenfor), betyder det, at systemet ikke er installeret endnu. Afbryd venligst forbindelsen til ssh og vent et par minutter for at logge på igen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Når `Welcome to Ubuntu 22.04.1 LTS` vises, er initialiseringen fuldført, og du kan fortsætte med følgende trin.

### [Valgfrit] Initialiser udviklingsmiljøet

Dette trin er valgfrit.

For nemheds skyld lægger jeg installationen og systemkonfigurationen af ​​ubuntu-software i [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Kør følgende kommando for at installere med et enkelt klik.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kinesiske brugere, brug venligst følgende kommando i stedet, og sproget, tidszonen osv. indstilles automatisk.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo muliggør IPV6

Aktiver IPV6, så SMTP også kan sende e-mails med IPV6-adresser.

rediger `/etc/sysctl.conf`

Rediger eller tilføj følgende linjer

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Følg op med [kontaktvejledningen: Tilføjelse af IPv6-forbindelse til din server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Rediger `/etc/netplan/01-netcfg.yaml` , tilføj et par linjer som vist i figuren nedenfor (Contabo VPS standard konfigurationsfil har allerede disse linjer, bare fjern kommentarer).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

`netplan apply` for at få den ændrede konfiguration til at træde i kraft.

Når konfigurationen er vellykket, kan du bruge `curl 6.ipw.cn` til at se ipv6-adressen på dit eksterne netværk.

## Klon ops for konfigurationslageret

```
git clone https://github.com/wactax/ops.soft.git
```

## Generer et gratis SSL-certifikat til dit domænenavn

Afsendelse af mail kræver et SSL-certifikat til kryptering og signering.

Vi bruger [acme.sh](https://github.com/acmesh-official/acme.sh) til at generere certifikater.

acme.sh er et open source automatiseret certifikatsigneringsværktøj,

Indtast konfigurationslageret ops.soft, kør `./ssl.sh` , og en `conf` mappe vil blive oprettet i **den øverste mappe** .

Find din DNS-udbyder fra [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , rediger `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Kør derefter `./ssl.sh 123.com` for at generere `123.com` og `*.123.com` certifikater for dit domænenavn.

Den første kørsel vil automatisk installere [acme.sh](https://github.com/acmesh-official/acme.sh) og tilføje en planlagt opgave til automatisk fornyelse. Du kan se `crontab -l` , der er sådan en linje som følger.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Stien til det genererede certifikat er noget som `/mnt/www/.acme.sh/123.com_ecc。`

Certifikatfornyelse vil kalde `conf/reload/123.com.sh` script, rediger dette script, du kan tilføje kommandoer såsom `nginx -s reload` for at opdatere certifikatcachen for relaterede applikationer.

## Byg SMTP-server med chasquid

[chasquid](https://github.com/albertito/chasquid) er en open source SMTP-server skrevet på Go-sproget.

Som en erstatning for de ældgamle mailserverprogrammer som Postfix og Sendmail er chasquid enklere og nemmere at bruge, og det er også nemmere for sekundær udvikling.

Kør `./chasquid/init.sh 123.com` vil blive installeret automatisk med et enkelt klik (erstat 123.com med dit afsendende domænenavn).

## Konfigurer e-mailsignatur DKIM

DKIM bruges til at sende e-mailsignaturer for at forhindre breve i at blive behandlet som spam.

Når kommandoen er kørt korrekt, bliver du bedt om at indstille DKIM-posten (som vist nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Du skal blot tilføje en TXT-post til din DNS (som vist nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Se servicestatus og logfiler

 `systemctl status chasquid` Se servicestatus.

Normal drift er som vist i figuren nedenfor

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` eller `journalctl -xeu chasquid` kan se fejlloggen.

## Omvendt konfiguration af domænenavn

Det omvendte domænenavn er for at tillade, at IP-adressen kan omsættes til det tilsvarende domænenavn.

Indstilling af et omvendt domænenavn kan forhindre e-mails i at blive identificeret som spam.

Når mailen er modtaget, vil den modtagende server udføre omvendt domænenavnsanalyse på IP-adressen på den afsendende server for at bekræfte, om den afsendende server har et gyldigt omvendt domænenavn.

Hvis den afsendende server ikke har et omvendt domænenavn, eller hvis det omvendte domænenavn ikke matcher IP-adressen på den afsendende server, kan den modtagende server genkende e-mailen som spam eller afvise den.

Besøg [https://my.contabo.com/rdns](https://my.contabo.com/rdns) og konfigurer som vist nedenfor

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Når du har indstillet det omvendte domænenavn, skal du huske at konfigurere fremadopløsningen af ​​domænenavnet ipv4 og ipv6 til serveren.

## Rediger værtsnavnet for chasquid.conf

Rediger `conf/chasquid/chasquid.conf` til værdien af ​​det omvendte domænenavn.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Kør derefter `systemctl restart chasquid` for at genstarte tjenesten.

## Backup conf til git repository

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

For eksempel sikkerhedskopierer jeg conf-mappen til min egen github-proces som følger

Opret et privat lager først

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Gå ind i conf-biblioteket og send til lageret

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Tilføj afsender

løb

```
chasquid-util user-add i@wac.tax
```

Kan tilføje afsender

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Bekræft, at adgangskoden er indstillet korrekt

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Efter tilføjelse af brugeren vil `chasquid/domains/wac.tax/users` blive opdateret, husk at indsende det til lageret.

## DNS tilføjer SPF-post

SPF (Sender Policy Framework) er en e-mailbekræftelsesteknologi, der bruges til at forhindre e-mailsvindel.

Det verificerer identiteten af ​​en e-mail-afsender ved at kontrollere, at afsenderens IP-adresse matcher DNS-registreringerne for det domænenavn, det hævder at være, hvilket forhindrer svindlere i at sende falske e-mails.

Tilføjelse af SPF-poster kan forhindre e-mails i at blive identificeret som spam så meget som muligt.

Hvis din domænenavneserver ikke understøtter SPF-type, skal du blot tilføje TXT-typepost.

For eksempel er SPF for `wac.tax` som følger

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF for `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Bemærk, at jeg har `include:_spf.google.com` her, det er fordi jeg vil konfigurere `i@wac.tax` som afsenderadresse i Google-postkassen senere.

## DNS-konfiguration DMARC

DMARC er forkortelsen af ​​(Domain-based Message Authentication, Reporting & Conformance).

Det bruges til at fange SPF-afvisninger (måske forårsaget af konfigurationsfejl, eller en anden udgiver sig for at være dig for at sende spam).

Tilføj TXT-post `_dmarc` ,

Indholdet er som følger

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Betydningen af ​​hver parameter er som følger

### p (Politik)

Angiver, hvordan man håndterer e-mails, der ikke bekræfter SPF (Sender Policy Framework) eller DKIM (DomainKeys Identified Mail). Parameteren p kan indstilles til en af ​​tre værdier:

* ingen: Der foretages ingen handling, kun bekræftelsesresultatet bliver tilbageført til afsenderen via e-mail-rapporteringsmekanismen.
* Karantæne: Læg den mail, der ikke har bestået bekræftelsen, i spam-mappen, men vil ikke afvise mailen direkte.
* afvis: Afvis direkte e-mails, der ikke bekræftes.

### fo (fejlmuligheder)

Angiver mængden af ​​information, der returneres af rapporteringsmekanismen. Den kan indstilles til en af ​​følgende værdier:

* 0: Rapporter valideringsresultater for alle meddelelser
* 1: Rapporter kun meddelelser, der ikke bekræftes
* d: Rapportér kun fejl i domænenavnsbekræftelsen
* s: rapporter kun SPF-verifikationsfejl
* l: Rapportér kun DKIM-verifikationsfejl

### rua & ruf

* rua (Reporting URI for Aggregate Reports): E-mailadresse til modtagelse af aggregerede rapporter
* ruf (Reporting URI for Forensic reports): e-mailadresse for at modtage detaljerede rapporter

## Tilføj MX-poster for at videresende e-mails til Google Mail

Fordi jeg ikke kunne finde en gratis virksomhedspostkasse, der understøtter universelle adresser (Catch-All, kan modtage alle e-mails sendt til dette domænenavn, uden begrænsninger på præfikser), brugte jeg chasquid til at videresende alle e-mails til min Gmail-postkasse.

**Hvis du har din egen betalte virksomhedspostkasse, skal du ikke ændre MX'en og springe dette trin over.**

Rediger `conf/chasquid/domains/wac.tax/aliases` , indstil videresendelsespostkasse

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` angiver alle e-mails, `i` er e-mail-adressepræfikset for den afsendende bruger oprettet ovenfor. For at videresende mail skal hver bruger tilføje en linje.

Tilføj derefter MX-posten (jeg peger direkte på adressen på det omvendte domænenavn her, som vist i første linje i figuren nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Når konfigurationen er fuldført, kan du bruge andre e-mail-adresser til at sende e-mails til `i@wac.tax` og `any123@wac.tax` for at se, om du kan modtage e-mails i Gmail.

Hvis ikke, så tjek chasquid-loggen ( `grep chasquid /var/log/syslog` ).

## Send en e-mail til i@wac.tax med Google Mail

Efter at Google Mail havde modtaget mailen, håbede jeg naturligvis at svare med `i@wac.tax` i stedet for i.wac.tax@gmail.com.

Besøg [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) og klik på "Tilføj en anden e-mailadresse".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Indtast derefter bekræftelseskoden modtaget af den e-mail, der blev videresendt til.

Endelig kan den indstilles som standard afsenderadresse (sammen med muligheden for at svare med samme adresse).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

På den måde har vi gennemført etableringen af ​​SMTP mailserveren og bruger samtidig Google Mail til at sende og modtage mails.

## Send en test-e-mail for at kontrollere, om konfigurationen er vellykket

Indtast `ops/chasquid`

Kør `direnv allow` at installere afhængigheder (direnv er blevet installeret i den tidligere one-key initialiseringsproces og en hook er blevet tilføjet til shellen)

så løb

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Betydningen af ​​parametrene er som følger

* bruger: SMTP-brugernavn
* pass: SMTP-adgangskode
* til: modtager

Du kan sende en test-e-mail.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Det anbefales at bruge Gmail til at modtage test-e-mails for at kontrollere, om konfigurationerne lykkes.

### TLS standard kryptering

Som vist i figuren nedenfor er der denne lille lås, som betyder, at SSL-certifikatet er blevet aktiveret.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Klik derefter på "Vis original e-mail"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Som vist i figuren nedenfor viser Gmails originale mailside DKIM, hvilket betyder, at DKIM-konfigurationen er vellykket.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Tjek Modtaget i overskriften på den originale e-mail, og du kan se, at afsenderadressen er IPV6, hvilket betyder, at IPV6 også er konfigureret med succes.
