---
layout: post
title:  Ulkoisen web-palvelimen määrittäminen weewx:iin
---
Weewx luo oletuksena nettisivunsa samalle koneelle, jolle se on asennettu.
Mikäli sivujen näyttämiseen halutaan käyttää toista konetta, voidaan se tehdä
käyttämällä StandardReportin lisäksi joko RSYNC- tai FTP-raporttia
`weewx.conf`:ssa, riippuen siitä millä tavalla sivut halutaan siirtää. Tässä
esimerkissä käytämme sivujen siirtämiseen rsync:iä ja SSH-avainparia
salasanattomaan tunnistautumiseen.

## SSH:n perusteet
SSH:ta eli Secure Shelliä käytetään nimensä mukaisesti ottamaan
komentoriviyhteys etäkoneeseen. SSH toimii asiakas-palvelin -periaatteella, eli
etäkoneella tulee olla käynnissä SSH-palvelin, jotta yhteys voidaan muodostaa.
Tunnistautuminen voidaan suorittaa perinteisesti salasanalla tai käyttämällä
avainparia. Avainparissa on se etu, että sitä on käytännössä mahdotonta murtaa.

Avainpariin kuuluu yksityinen ja julkinen avain. Julkinen avain lähetetään
koneelle, johon halutaan ottaa SSH-yhteys. Yhteys otetaan käyttämällä yksityistä
avainta, joka voi olla salasanasuojattu. Julkisen avaimen päätyminen vääriin
käsiin ei ole niinkään vaarallista, mutta yksityinen avain tulee pitää hyvässä
tallessa, varsinkin jos sitä ei ole suojattu salasanalla.

Julkinen avain voidaan ajatella lukkona, johon vain yksityinen avain sopii.
Mikäli yksityisessä avaimessa ei ole salasanaa, teoriassa kuka tahansa käyttää
avainta etäkoneelle kirjautumiseen. Käytännössä SSH-kirjautumista voidaan
rajoittaa esimerkiksi määrittämällä julkiseen avaimeen IP-osoitteet, joista
yhteys voidaan ottaa. Lisäksi julkisen avaimen käyttö voidaan rajoittaa johonkin
tiettyyn komentoon.

## Etäkoneen asetukset
Etäkoneella, joka tulee toimimaan web-palvelimena, tulee olla käynnissä
SSH-palvelin sekä web-palvelin, jolla moduuli `mod_userdir` on käytössä. Lisäksi
koneen palomuurin tulee sallia liikenne SSH-palvelimen porttiin, joka on
oletuksena 22. Koska käytämme rsync:iä sivujen lähettämiseen, pitää rsync olla
asennettuna. Koneella tulee olla tili, jolla weewx voi kirjautua SSH:n kautta.
Yksinkertaisuuden vuoksi se kannattaa nimetä myös weewx:ksi, koska SSH käyttää
oletuksena paikallisen käytäjän nimeä etäkoneelle kirjautuessa.

Mikäli etäpalvelimelle päästään fyysisesti kirjautumaan, voidaan avainparin
julkinen avain kopioida esimerkiksi USB-tikulla. Muutoin voidaan
SSH-palvelimelta tilapäisesti sallia salasanalla kirjautuminen, jotta julkinen
avain saadaan kopioitua etäkoneelle. Jotta kirjautuminen salasanalla onnistuu,
pitää weewx-käyttäjällä olla salasana. Kun avainpohjainen kirjautuminen on
todettu toimivaksi, voidaan salasanalla kirjautuminen kieltää. Tämä käytännössä
estää luvattomat SSH-kirjautumiset kunhan yksityinen avain on turvassa.

## Paikallisen koneen asetukset
Paikallisella koneella luodaan SSH-avainpari, lähetetään julkinen avain
etäkoneelle, todetaan SSH-yhteys toimivaksi ja lopuksi määritellään weewx
lähettämään nettisivunsa etäpalvelimelle. Avainpari luodaan alla olevalla
komennolla. Komento kysyy yksityisen avaimen tiedostonimeä sekä salasanaa.
Avaimen nimen voi halutessaan muuttaa esimerkiksi `id_weewx`:ksi. Salasanan
pitää olla tyhjä, jotta weewx voi toimia itsenäisesti. Julkinen avain
tallennetaan .pub-päätteellä, joten sen nimi voi olla esimerkiksi
`id_weewx.pub`.

``` bash
# -t: avaintyyppi
# -C: avaimen kuvaus (comment)
ssh-keygen -t ed25519 -C "weewx@rpi"
```

Julkinen avain pitää tämän jälkeen lähettää etäkoneelle SSH-yhteyden kautta
seuraavalla komennolla.

``` bash
# -i: julkisen avaimen sijainti
# <IP>: etäkoneen IP-osoite
ssh-copy-id -i ~/.ssh/id_weewx.pub <IP>
```

Tämän jälkeen etäkoneelle pitäisi voida kirjautua yksityisellä avaimella.

``` bash
# -i: yksityisen avaimen sijainti
# <IP>: etäkoneen IP-osoite
ssh -i ~/.ssh/id_weewx <IP>
```

Mikäli yhteys toimii, voidaan yhteydelle luoda alias, jotta yhteys voidaan luoda
esimerkiksi komennolla `ssh palvelin`. Tämä tapahtuu muokkaamalla tiedostoa
`~/.ssh/config` ja lisätä sinne seuraavanlaiset rivit. *Host* määrittelee
aliaksen nimen, joten se voi olla muukin kuin `palvelin`.

```
Host palvelin
    Hostname <IP>
    IdentityFile ~/.ssh/id_weewx
```

Aliaksen toiminta kannattaa varmistaa komennolla `ssh palvelin`.

Lopuksi määritellään weewx lähettämään nettisivunsa etäpalvelimelle. Lähetys
tapahtuu oletuksena 5 minuutin välein heti nettisivujen luonnin jälkeen.
Tiedostossa `~/weewx.conf` pitää muokata `[StdReports]`-osion sisällä olevaa
`RSYNC`-raporttia. Raportissa on 3 kommentoitua riviä, jotka määrittävät
etäpalvelimen nimen, etäpalvelimella olevan kansion jonne sivut sijoitetaan sekä
käyttäjän, jolla etäpalvelimelle kirjaudutaan. Lisäksi *delete*-muuttuja
määrittelee, poistetaanko etäkansiosta ylimääräiset tiedostot. Oletuksena asetus
on pois päältä, koska sillä on mahdollista aiheuttaa vahinkoa, jos esimerkiksi
etäkansio on määritelty väärin. Toisaalta jos asetus ei ole päällä, voi
etäkansioon kertyä paljon ylimääräisiä tiedostoja. Kannattaa pitää asetus
aluksi pois päältä ja laittaa asetus päälle vasta kun on todettu synkronoinnin
toimivuus.

Alla on esimerkki tarvittavista riveistä RSYNC-raportissa. Muuttujassa *delete*
arvo 0 tarkoittaa asetuksen olevan pois käytöstä. `public_html` on kansio, josta
web-palvelin hakee käyttäjien nettisivut, ja näyttää ne esimerkiksi osoitteessa
`127.0.0.1/~weewx`.

``` python
server = palvelin
path = public_html
user = weewx
delete = 0
```

## SSH:n tietoturvan parantaminen
Käyttämämme yksityinen avain ei ole salattu salasanalla, joten sen joutuminen
vääriin käsiin voi antaa vieraalle taholle mahdollisuuden ajaa haluamiaan
komentoja miltä koneelta tahansa. Voimme pienentää ja jopa poistaa tämän uhan
määrittämällä julkiseen avaimeen paikallisen koneen IP-osoitteen ja
rsync-komennon, jonka weewx ajaa. Tämän jälkeen yksityisellä avaimella voidaan
ajaa vain weewx:n suorittama rsync-komento vain paikallisen koneen
IP-osoitteesta käsin.

Julkinen avain sijaitsee etäkoneella tiedostossa `~/.ssh/authorized_keys`.
Tiedostossa voi olla muitakin avaimia. Luomamme julkinen avain voi näyttää
seuraavanlaiselta:

``` bash
# avaintyyppi avain kommentti
ssh-ed25519 AAAA... weewx@rpi
```

Avaimen parametrit, kuten sallitut IP-osoitteet ja komennot, tulee kirjoittaa
rivin alkuun. Parametrit erotetaan toisistaan pilkulla. Alla on esimerkki, jossa
vain `ls`-komento voidaan suorittaa ja vain osoitteesta `1.2.3.4`.

``` bash
from="1.2.3.4",command="ls" ssh-ed25519 AAAA... weewx@rpi
```

Paikallisen koneen ulkoisen IP-osoitteen  saa selville esimerkiksi komennolla
`curl https://icanhazip.com`.

Rsync-komento, jonka weewx ajaa tallentuu SSH-istunnon ajaksi
ympäristömuuttujaan *SSH_ORIGINAL_COMMAND*. Voimme saada sen selville luomalla
scriptin, joka tallentaa komennon etäkoneelle, ja määrittämällä sen tilapäisesti
avaimen komennoksi. Kun weewx sitten ajetaan, tallentuu rsync-komento
haluamaamme tiedostoon. Alla on skriptin sisältö.

``` bash
echo $SSH_ORIGINAL_COMMAND > /tmp/rsync.cmd
```

Se voidaan tallentaa esimerkiksi tiedostoon `/home/weewx/ssh-cmd`. Sille pitää
muistaa antaa suoritusoikeudet komennolla `chmod u+x /home/weewx/ssh-cmd`. Tämän
jälkeen skripti voidaan lisätä avaimen parametreihin:

``` bash
from="1.2.3.4",command="/home/weewx/ssh-cmd" ssh-ed25519 AAAA... weewx@rpi
```

Nyt weewx voidaan ajaa paikallisella koneella, jolloin se ainoastaan suorittaa
tekemämme skriptin. Kun skripti on kerran ajettu, löytyy weewx:n rsync-komento
skriptin määrittämästä tiedostosta. Tämä rsync-komento voidaan nyt määritellä
avaimen parametreihin:

``` bash
from="1.2.3.4",command="rsync ..." ssh-ed25519 AAAA... weewx@rpi
```

Avaimen kaikki parametrit löytyvät `sshd`:n man-sivusta osiosta *AUTHORIZED_KEYS
FILE FORMAT*. Sivun saa näkyviin komennolla `man sshd`. Osiossa mainitut
*no*-alkuiset parametrit voidaan myös ottaa käyttöön huoletta seuraavasti:

``` bash
from="1.2.3.4",command="ls",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-X11-forwarding ssh-ed25519 AAAA... weewx@rpi
```

Tämän lisäksi etäkoneen palomuuri voidaan määritellä esimerkiksi siten, että
SSH-yhteydet ovat sallittuja vain paikallisesta koneesta.
