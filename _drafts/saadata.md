---
layout:     post
title:      Säädatan jakaminen internetiin
---
## Projektin tavoite
Tavoitteena on [Fine Offsetin WH1080][wh1080] -sääasemaa sekä [Raspberry Pi][rpi]:tä käyttämällä luoda nettisivut, jotka näyttävät ja keräävät paikallista säädataa. Raspberry Pi yhdistetään sääaseman näyttöön USB-kaapelilla, ja [weewx][weewx] huolehtii säädatan keräämisestä Pi:lle. Pi:n käyttöjärjestelmänä toimii [Raspbian][raspbian]:in Jessie-versio ja web-palvelimena [Lighttpd][lighttpd].

## Raspberry Pi
### Käynnistys
Oletuksena Pi käynnistyy graafiseen tilaan, mikä ei ole tarpeellista mikäli Pi toimii vain web-palvelimena. Pi voidaan määrittää käynnistymään komentorivitilaan ajamalla komento *raspi-config* root-käyttäjän oikeuksilla, menemällä kohtaan *3 Boot Options*, valitsemalla joko *B1* tai *B2*, ja valitsemalla *&lt;Ok&gt;*.

### Muistinjako
Raspi-configissa voidaan myös määritellä miten Pi jakaa keskusmuistinsa prosessorin ja näytönohjaimen kesken. Web-palvelin ei tarvitse näytönohjainta toimiakseen, varsinkaan jos Pi:tä hallitaan SSH:n välityksellä, joten prosessorille voidaan antaa maksimimäärä muistia. Muistinjakoa säädetään valikosta *9 Advanced Options* kohdasta *A3 Memory Split*, jonne annetaan näytönohjaimelle varattavan muistin määrä, joka voi olla minimissään 16 megatavua.

## Palomuuri
Oletuksena Pi:n palomuuri sallii kaiken liikenteen. Pi käyttää palomuurina Linuxiin sisäänrakennettua iptables:ia. Sen konfigurointi onnistuu käyttämällä *iptables*-komentoa, mutta käytämme tässä tapauksessa [ufw][ufw]:ta, joka on edustaohjelma iptablesille. Ufw:n asennus ja konfigurointi tehdään alla olevilla komennoilla. Komennot vaativat root-käyttäjän oikeudet.

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
# laitetaan päälle käyttäjien omat nettisivut
# esim. 127.0.0.1/~user
lighty-enable-mod userdir
# asetetaan lighttpd käynnistymään aina käynnistyksen yhteydessä
systemctl enable lighttpd
# käynnistetään lighttpd
systemctl start lighttpd
```

Oletuksena weewx:n tuottamat nettisivut näkyvät osoitteessa `127.0.0.1/~weewx`. Lighttpd:ssä osoitteille voidaan määritellä aliaksia, jolloin weewx:n nettisivuille voi päästä myös esimerkiksi osoitteella `127.0.0.1/weather`. Alla on esimerkki `/weather`-aliaksen luomisesta.

``` bash
# luodaan tarvittava konfiguraatiotiedosto
echo 'alias.url += ("/weather" => "/home/weewx/public_html")' > /etc/lighttpd/conf-available/90-weewx-alias.conf
# otetaan kyseinen tiedosto käyttöön linkittämämällä se oikeaan paikkaan
# !$ tarkoittaa edellisen komennon viimeistä argumenttia
ln -s !$ /etc/lighttpd/conf-enabled/
# uudelleenkäynnistetään lighttpd
systemctl restart lighttpd
```

## weewx
### Asennus
Weewx:n tehtävänä on muun muassa noutaa sääasemalta säätietoja, arkistoida niitä ja luoda niistä erilaisia raportteja, esimerkiksi nettisivuja. Weewx voi myös lähettää säätietoja verkon sääpalveluihin, kuten [Weather Underground][wug]:iin. Weewx:n voi myös määritellä käyttämään jotain toista web-palvelinta sivujen näyttämiseen.

Weewx on saatavilla Raspbianin pakettienhallinnasta, mutta sen oletusasetukset eivät ole parhaat mahdolliset. Raspbianin paketti esimerkiksi toimii oletuksena root-käyttäjän oikeuksilla eikä siitä löydy vielä systemd-palvelua. Haemme weewx:n lähdekoodin käyttämällä [git][git]-versionhallintajärjestelmää ja asennamme weewx:n erilliselle weewx-käyttäjälle. Emme käytä mahdollisesti epävakaata kehitysversiota weewx:stä, vaan valitsemme uusimman vakaan version, mikä on kirjoitushetkellä 3.2.1. Hyödynnämme Raspbianin pakettienhallintaa hakemalla pelkästään weewx-paketin vaatimat paketit asentamatta itse weewx:iä.

Asentamisen jälkeen weewx pitää vielä konfiguroida ajamalla *wee_config*-komento ja muokkaamalla `weewx.conf`-tiedostoa. Wee\_config pitää huolen tärkeimmistä asetuksista, mutta muutama asetus pitää käydä itse muuttamassa.

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

### Automaattinen sivun päivitys
Oletuksena weewx:n tuottama nettisivu ei päivitä itseään automaattisesti. Tämän voi muuttaa lisäämällä käytössä olevan teeman `index.hmtl.tmpl`-tiedoston *&lt;head&gt;*-osioon alla olevan rivin, joka päivittää sivun 150 sekunnin eli 2,5 minuutin välein.

``` html
<meta http-equiv="refresh" content="150">
```

### Palvelu
Weewx:n on tarkoitus olla yhtäjaksoisesti päällä, joten on järkevää ajaa sitä palveluna. Jessie on Raspbianin ensimmäinen julkaisu, joka käyttää systemd:tä init-ohjelmana. Init huolehtii muun musssa käyttöjärjestelmän käynnistyksestä ja palvelujen ajamisesta. Aiemmat SysV-initiin tehdyt palvelutiedostot eivät toimi systemd:ssä, vaan se käyttää omia yksinkertaisempia palvelutiedostoja. Weewx:n lähdekoodin mukana tulee systemd:lle tehty palvelutiedosto, `weewx.service`, jota voimme käyttää pienin muutoksin. Tiedosto on kansiossa `/home/weewx/weewx/util/systemd/`. Tiedostosta pitää muuttaa seuraavat asiat:

- *User*- ja *Group*-rivien pitää olla kommentoimatta
- Pid-tiedoston sijainti, joka mainitaan *ExecStart*- ja *PIDFile*-riveillä, pitää muuttaa seuraaksi: `/home/weewx/run/weewx.pid`

Tiedosto pitää tämän jälkeen linkittää oikeaan kansioon rootin oikeuksilla:

``` bash
ln -s /home/weewx/weewx/util/systemd/weewx.service /etc/systemd/system
```

Palvelu käynnistetään komennolla `systemctl start weewx` ja komento `systemctl enable weewx` määrittää palvelun käynnistymään Pi:n käynnistyksen yhteydessä. Palvelu kannattaa käynnistää vasta myöhemmin, kun ohjeen muut vaiheet on suoritettu.

## Laiteasetukset
### Oikeudet
Oletuksena weewx-käyttäjällä ei ole riittävästi oikeuksia käsitellä sääasemaan laitetiedostoa. Voimme määritellä laitetiedostolle oikeat oikeudet käyttämällä udev-sääntöä. Weewx:n lähdekoodiin on lisätty Fine Offsetin sääasemille sopiva udev-sääntö, joka antaa kaikille käyttäjille kirjoitusoikeudet sääasemaan. Tämän säännön voi ottaa käyttöön linkittämällä sen oikeaan kansioon rootin oikeuksilla alla olevalla käskyllä.

``` bash
ln -s /home/weewx/weewx/util/udev/rules.d/fousb.rules /etc/udev/rules.d/90-wh1080.rules
```

Mikäli weewx-ryhmän ulkopuolisille käyttäjille ei halua antaa kirjoitusoikeutta, voi säännöstä muokata seuraavanlaisen.

``` bash
SUBSYSTEM=="usb", ATTRS{idVendor}=="1941", ATTRS{idProduct}=="8021", MODE="0664", GROUP="weewx"
```

### Arkistointiväli
Oletuksena WH1080 tallentaa säädataa 30 minuutin välein, kun taas weewx toimii 5 minuutin intervallilla. Alla olevalla komennolla voimme määritellä sääaseman toimimaan samalla intervallilla kuin weewx. Sääasema tulee luonnollisesti olla kytkettynä Pi:hin, jotta komento voidaan ajaa.

``` bash
/home/weewx/bin/wee_device --set-interval 5
```


## anything-sync-daemon
Raspberry Pi käyttää tiedon tallennukseen MicroSD-korttia, jolla on rajallinen elinikä. Voimme vähentää muistikortin käyttöä tallentamalla tietoa Pi:n RAM-muistiin käyttämällä [anything-sync-daemon][asd]ia, lyhyemmin asd:ia, joka pitää haluttujen kansioiden sisällön RAM-muistissa, ja varmuuskopioi ne muistikortille säännöllisin väliajoin. Weewx päivittää oletuksena viiden minuutin välein oman arkistonsa kansioon `/home/weewx/archive`, ja sen pohjalta luodun nettisivun kansioon `/home/weewx/public_html`. Kun nämä kaksi kansiota tallennetaan RAM-muistiin, weewx:n muistikortille kirjoittama datamäärä pienenee huomattavasti.

Ohjelmaa ei ole paketoitu Raspbianille, joten asennamme sen manuaalisesti samaan tapaan kuten weewx:n alla olevien ohjeiden mukaan. Asennus tehdään pi-käyttäjätunnuksella.

``` bash
# haetaan asd:in lähdekoodi
git clone 'https://github.com/graysky2/anything-sync-daemon.git'
# siirrytään lähdekoodin kansioon
cd anything-sync-daemon
# listataan lähdekoodin tagit
git tag --list
# siirrytään käyttämään versiota v5.76
git checkout v5.76
# asennetaan asd
make
sudo make install-systemd-all
```

Asennuksen jälkeen asd pitää konfiguroida, mikä tapahtuu muokkaamalla sen konfiguraatiotiedostoa `/etc/asd.conf`. Muuttuja *VOLATILE* määrittää kansion, jonka järjestelmä on liittänyt RAM-muistiin ja *WHATTOSYNC* taas sisältää listan kansioista, joiden sisältö halutaan säilöä RAM-muistiin. Raspbianissa `/tmp` ei ole oletuksena liitetty RAM-muistiin. Asian voi korjata tiedostossa `/etc/default/tmpfs` poistamalla kommentin riviltä `#RAMTMP=yes`. Pi täytyy käynnistää uudelleen, jotta muutos tulee voimaan. Muuttuja *WHATTOSYNC* voidaan määritellään esimerkiksi seuraavasti:

``` bash
WHATTOSYNC=('/home/weewx/archive' '/home/weewx/public_html')
```

Kun asd on konfiguroitu, sen voi käynnistää komennolla `systemctl start asd`. Komennolla `systemctl enable asd` määritellään asd käynnistymään aina Pi:n käynnistyessä.

## weewx:n käynnistys
Lopuksi on aika käynnistää weewx ja varmistaa, että se toimii oikein. Komennoilla `systemctl start weewx`, `systemctl stop weewx` ja `systemctl restart weewx` käynnistetään, pysäytetään ja uudelleenkäynnistetään weewx. Komennolla `systemctl reload weewx` weewx lukee `weewx.conf`:in uudelleen.

Komento `journalctl -u weewx` näyttää weewx:n lokin. Valitsimella `-b` `journalctl` näyttää viimeisen käynnistyksen jälkeen tehdyt lokimerkinnät. Valitsimella `--since` voidaan katsella tietyn ajan jälkeen tehtyjä lokeja; esimerkiksi `--since=today` näyttää tämän päivän lokit, ja `--since=-1h` näyttää viimeisen tunnin lokit.

## Extra: Sofaskin
Weewx:lle saatavilla useita valmiita [teemoja][wx-skins], kuten [Sofaskin][sofaskin]. Teemat asennetaan sijoittamalla teeman kansio teemoille tarkoitettuun hakemistoon `/home/weewx/skins`, ja otetaan käyttöön vaihtamalla teemaa `weewx.conf`:ista. Jokaisella teemalla on kansiossaan oma konfiguraatiotiedosto `skin.conf`. Teemaa voi muokata joko muokkaamalla `skin.conf`:ia tai `weewx.conf`:ia. Alla olevassa esimerkissä asennetaan Sofaskin ja muokataan sitä `weewx.conf`:ista käsin.

``` bash
cd ~/skins
# tehdään Sofaskinille oma kansio
mkdir Sofaskin
# siirrytään sinne
cd !$
# ladataan Sofaskin itse nimeämäämme tiedostoon
curl "http://en.blauesledersofa.de/?wpdmdl=18" -o sofaskin.zip
# puretaan ladattu zip-paketti
unzip !$
# mikäli zip-pakettia ei halua säilyttää, voi sen poistaa
rm !$
```

Tämän jälkeen muokataan `weewx.conf`:ia. Osiossa `[StdReport]` määritellään, minkälaisia nettisivuja, weewx:n termein raportteja, weewx tuottaa. Oletuksena weewx tuottaa yhden nettisivun StandardReport-raportin mukaan. `[StdReport]`-osioon voi lisätä raportteja mielensä mukaan; esimerkiksi yhden normaalin nettisivun, toisen brittiläisiä yksiköitä käyttävän sivun ja kolmannen eri teemaa käyttävän sivun.

Raportissa teema määritellään *skin*-muuttujalla. Jos määrittelyä ei tee, käyttää weewx raportissa *Standard*-teemaa. Raportin konfiguroinnin yhteydessä voidaan muuttaa myös teeman asetuksia, jotka sijaitsevat teeman `skin.conf`-tiedostossa. Sofaskinin `skin.conf`:ssa esimerkiksi on rivi, joka määrittelee sivun tekijän:

``` python
[Extras]
    you = Sven
```

Tämän asetuksen voi muuttaa `weewx.conf`:sta käsin seuraavalla tavalla:

``` python
[StdReport]
    [[StandardReport]]
        [[[Extras]]]
            you = nimi
```

Sofaskinillä on saksalainen tekijä, joten joidenkin yksiköiden nimet ovat oletuksena saksankielisiä. Lisäksi tuulen nopeuden yksikkönä käytetään km/h.  Nämäkin asiat voidaan korjata `weewx.conf`:issa. Alla on esimerkkiraportti `weewx.conf`:ista, jossa yllä mainitut asiat on korjattu.

``` python
[StdReport]
    [[StandardReport]]
        skin = Sofaskin
        [[[Extras]]]
            you = nimi
        [[[Units]]]
            [[[[Groups]]]]
                group_speed = meter_per_second
                group_speed2 = meter_per_second2
            [[[[Labels]]]]
                knot = " knots"
                knot2 = " knots"
                meter = " meters"
                day = " Day", " Days"
                hour = " Hour", " Hours"
                minute = " Minute", " Minutes"
                second = " Second", " Seconds"
```

Oletuksena Sofaskin näyttää myös sää- ja salamatutkakuvia. Ne saa pois käytöstä
kommentoimalla *radar_*- ja *lightning*-alkuiset rivit `skin.conf`:in
`[Extras]`-osiosta.

## Linkkejä
- [weewx:n oppaat][wx-docs]
- [weewx:n wiki][wx-wiki]

[wh1080]: http://www.foshk.com/Weather_Professional/WH1080.htm
[rpi]: https://www.raspberrypi.org/
[weewx]: http://weewx.com/
[raspbian]: https://raspbian.org/
[lighttpd]: http://www.lighttpd.net/
[ufw]: https://launchpad.net/ufw
[wug]: http://www.wunderground.com/
[git]: https://git-scm.com/
[asd]: https://github.com/graysky2/anything-sync-daemon
[wx-skins]: https://github.com/weewx/weewx/wiki#skins
[sofaskin]: http://en.blauesledersofa.de/2015/03/flat-template-for-weewx
[wx-docs]: http://weewx.com/docs.html
[wx-wiki]: https://github.com/weewx/weewx/wiki
