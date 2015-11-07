---
layout:     post
title:      Säädatan jakaminen internetiin
categories: school
tags:       weather
---
## Projektin tavoite
Tavoitteena on [Fine Offsetin WH1080][wh1080] -sääasemaa sekä [Raspberry Pi][rpi]:tä käyttämällä luoda web-sivut, jotka näyttävät ja keräävät paikallista säädataa. Raspberry Pi yhdistetään sääaseman näyttöön USB-kaapelilla, ja [weewx][weewx] huolehtii säädatan keräämisestä Pi:lle. Pi:n käyttöjärjestelmänä toimii [Raspbian][raspbian]:in Jessie-versio ja web-palvelimena [Lighttpd][lighttpd].

## Raspberry Pi:n asetukset
Oletuksena Pi käynnistyy graafiseen tilaan, mikä ei ole tarpeellista mikäli Pi toimii vain web-palvelimena. Pi voidaan määrittää käynnistymään komentorivitilaan ajamalla komento *raspi-config* root-käyttäjän oikeuksilla, menemällä kohtaan *3 Boot Options*, valitsemalla joko *B1* tai *B2*, ja valitsemalla *Ok*.

Raspi-configissa voidaan myös määritellä miten Pi jakaa keskusmuistinsa prosessorin ja näytönohjaimen kesken. Web-palvelin ei tarvitse näytönohjainta toimiakseen, varsinkaan jos Pi:tä hallitaan SSH:n välityksellä, joten prosessorille voidaan antaa maksimimäärä muistia. Muistinjakoa säädetään valikosta *9 Advanced Options* kohdasta *A3 Memory Split*, jonne annetaan näytönohjaimelle varattavan muistin määrä, joka voi olla minimissään 16 megatavua.

## Palomuuri
Oletuksena Pi:n palomuuri sallii kaiken liikenteen. Pi käyttää palomuurina Linux:iin sisäänrakennettua iptables:ia. Sen konfigurointi onnistuu käyttämällä *iptables*-komentoa, mutta käytämme tässä tapauksessa [ufw][ufw]:ta, joka on edustaohjelma iptablesille. Ufw:n asennus ja konfigurointi tehdään alla olevilla komennoilla. Komennot vaativat root-käyttäjän oikeudet.

``` bash
# asennetaan ufw
apt-get install ufw
# sallitaan ssh-yhteydet ja kirjataan jokainen yhteys logiin
ufw allow log ssh
# sallitaan web-liikenne
ufw allow http
# laitetaan palomuuri päälle
ufw enable
```

## Web-palvelin
Lighttpd asennetaan ja konfiguroidaan seuraavilla root-oikeuksin ajettavilla komennoilla.

``` bash
# asennetaan lighttpd
apt-get install lighttpd
# laitetaan päälle käyttäjien omat web-sivut
# esim. 127.0.0.1/~user
lighty-enable-mod userdir
# asetetaan lighttpd käynnistymään aina käynnistyksen yhteydessä
systemctl enable lighttpd
# käynnistetään lighttpd
systemctl start lighttpd
```

Oletuksena weewx:n tuottamat web-sivut näkyvät osoitteessa `127.0.0.1/~weewx`. Lighttpd:ssä osoitteille voidaan määritellä aliaksia, jolloin weewx:n web-sivuille voi päästä myös esimerkiksi osoitteella `127.0.0.1/weather`. Alla on esimerkki /weather-aliaksen luomisesta.

``` bash
# luodaan tarvittava konfigurointitiedosto
echo 'alias.url += ("/weather" => "/home/weewx/public_html")' > /etc/lighttpd/conf-available/90-weewx-alias.conf
# otetaan kyseinen tiedosto käyttöön linkittämämällä se oikeaan paikkaan
# !$ tarkoittaa edellisen komennon viimeistä argumenttia
ln -s !$ /etc/lighttpd/conf-enabled/
# uudelleenkäynnistetään lighttpd
systemctl restart lighttpd
```

## Weewx
Weewx:n tehtävänä on muun muassa noutaa sääasemalta säätietoja, arkistoida niitä ja luoda niistä erilaisia raportteja, esimerkiksi web-sivuja. Weewx voi myös lähettää säätietoja verkon sääpalveluihin, kuten [Weather Underground][wunderground]:iin. Weewx:n voi myös määritellä käyttämään jotain toista web-palvelinta sivujen näyttämiseen.

Weewx on saatavilla Raspbianin pakettienhallinnasta, mutta sen oletusasetukset eivät ole parhaat mahdolliset. Raspbianin paketti esimerkiksi toimii oletuksena root-käyttäjän oikeuksilla eikä siitä löydy vielä systemd-palvelua. Haemme weewx:n lähdekoodin käyttämällä [git][git]-versionhallintajärjestelmää ja asennamme weewx:n erilliselle weewx-käyttäjälle. Emme käytä mahdollisesti epävakaata kehitysversiota weewx:stä, vaan valitsemme uusimman vakaan version, mikä on kirjoitushetkellä 3.2.1. Hyödynnämme Raspbianin pakettienhallintaa hakemalla pelkästään weewx-paketin vaatimat paketit asentamatta itse weewx:iä.

Asentamisen jälkeen weewx pitää vielä konfiguroida ajamalla *wee_config*-komento ja muokkaamalla *weewx.conf*-tiedostoa. wee_config pitää huolen tärkeimmistä asetuksista, mutta muutama asetus pitää käydä itse muuttamassa.

Alla on lista weewx:n asennukseen tarvittavista komennoista.

``` bash
# asennetaan weewx:n riippuvuudet
aptitude install '~R weewx'
# lisätään weewx-käyttäjä ja luodaan sille kotihakemisto
useradd --create-home weewx
# vaihdetaan weewx-käyttäjälle
su - weewx

# haetaan weewx:n lähdekoodi
git clone 'https://github.com/weewx/weewx.git'
# siirrytään lähdekoodin kansioon
cd weewx
# weewx-projekti merkitsee versiot käyttämällä gitin tageja
# ensin haemme listan kaikista tageista
git tag --list
# ja sitten vaihdamme haluamaamme tagiin
git checkout v3.2.1
# asennetaan weewx weewx-käyttäjän kotihakemistoon
./setup.py build
./setup.py install

# ajetaan wee_config, joka kysyy sääaseman perustiedot
# ja luo näiden tietojen pohjalta weewx.conf-tiedoston
cd
bin/wee_config --install --dist-config=weewx/weewx.conf --output=weewx.conf

# muokataan weewx.conf:ia
nano weewx.conf
# muokkaa osiota [Station]:
#   aseta maanantai viikon aloituspäiväksi:
#     week_start = 0
#
# muokkaaa osiota [FineOffsetUSB]:
#   aseta laitteen malli:
#     model = WH1080
```

## Laiteoikeudet
Oletuksena weewx-käyttäjällä ei ole riittävästi oikeuksia käsitellä sääasemaan laitetiedostoa. Voimme määritellä laitetiedostolle oikeat oikeudet käyttämällä udev-sääntöä. Weewx:n lähdekoodiin on lisätty Fine Offsetin sääasemille sopiva udev-sääntö, joka antaa kaikille käyttäjille kirjoitusoikeudet sääasemaan. Tämän säännön voi ottaa käyttöön linkittämällä sen oikeaan kansioon rootin oikeuksilla alla olevalla käskyllä.

``` bash
ln -s /home/weewx/weewx/util/udev/rules.d/fousb.rules /etc/udev/rules.d/90-wh1080.rules
```
Mikäli weewx-ryhmän ulkopuolisille käyttäjille ei halua antaa kirjoitusoikeutta, voi säännöstä muokata seuraavanlaisen.

``` bash
SUBSYSTEM=="usb", ATTRS{idVendor}=="1941", ATTRS{idProduct}=="8021", MODE="0664", GROUP="weewx"
```

## Palvelu
Weewx:n on tarkoitus olla yhtäjaksoisesti päällä, joten on järkevää ajaa sitä palveluna. Jessie on Raspbianin ensimmäinen julkaisu, joka käyttää systemd:tä init-ohjelmana. systemd huolehtii muun musssa käyttöjärjestelmän käynnistyksestä ja palvelujen ajamisesta. Aiemmat SysV-initiin tehdyt palvelutiedostot eivät toimi systemd:ssä, vaan se käyttää omia yksinkertaimpia palvelutiedostoja. Weewx:n lähdekoodin mukana tulee systemd:lle tehty palvelutiedosto, jota voimme käyttää pienin muutoksin. Tiedosto on kansiossa `/home/weewx/weewx/util/systemd/`. Tiedostosta pitää muuttaa seuraavat asiat:

- User- ja Group-rivien pitää olla kommentoimatta
- Pid-tiedoston sijainti, joka mainitaan ExecStart- ja PIDFile-riveillä, pitää muuttaa seuraaksi: `/home/weewx/run/weewx.pid`


[wh1080]: http://www.foshk.com/Weather_Professional/WH1080.htm
[rpi]: https://www.raspberrypi.org/
[weewx]: http://weewx.com/
[raspbian]: https://raspbian.org/
[lighttpd]: http://www.lighttpd.net/
[ufw]: https://launchpad.net/ufw
[wunderground]: http://www.wunderground.com/
[git]: https://git-scm.com/
