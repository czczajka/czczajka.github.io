---
layout: post
title:  "Zdalne aktualizacje oprogramowania z narzędziem SWUpdate"
date:   2022-11-22 00:12:24 +0100
categories: jekyll update
---

## Kilka słów na początek...
Wyobraz sobie ze wlasnie kupiles nowy sachochod. Nagle po dwoch miesiacach zostajesz pilnie wezwany do serwisu poniewaz Twoj nowy amochod jest wadliwy i musi dostac aktualizacje oprogramowania aby moc byc ponownie bezpiecznie uzytkowany. Nie jest jeszcze tak zle, wiele produktow nie ma wcale mozliwosci aktualizacji oprograwoania przez caly ich cykl zycia ale mogloby byc lepiej.

Co w przypku kiedy aktualizacje mozna by wykonac zdalnie a Ty nawet moglbys o niej nie wiedziec i poprostu cieszyc sie swoim samochodem. Brzmi rozsadnie, nieprawdaz ? Tak to dziala w Twoim telefonie, nikt Cie nie wzywa na akcje serwisowe a caly czas dostajesz wszelakie aktualizacje.

W atrykule chcialbym poruszyc temat zdalnych aktualizacji urzadzen Linux Embedded. Jest to artykul raczej dla osob poczatkujacych. Przedstawione i omowione zostana w nim rozne startegie aktualizacji oraz przyblizone zostanie narzedzie SWUpdate. Na koncu bedzie krotki przyklad  prostej aktualizacji płytki raspberry pi 4 z wykorzystaniem SWUpdate.

Czasem z roznych przyczyn urzadzenie Linux Embedded nie ma polaczenia z siecia Internet. W takim wypadku nie da się przeprowadzić aktualizacji zdalnie tylko trzeba użyć fo tego lokalnego interfejsu (USB, SD). W artykule skupie sie wyłącznie  na zdalnych aktualizacjach aczkowliek w przypaku większości poruszonych rozważań medium transmisyjne aktualizacji nie ma większego znaczenia.


## Kilka słów o projekcie SWUpdate
SWUpdate jest to 'Linux Update agent', który za główne zadanie ma Nam pomóc w przeprowadzeniu aktualizacji systemu Embedded Linux. Może zostać zainstalowany za pomocą komendy 'apt' bądz zbudowany ze źródeł. Projekt daje bardzo dużo różnych możliwości i dzięki niemu nie musimy wymyślać wsztkiego od nowa a możemy skorzystać z gotowego, dobrze przetestowanego narzędzia

Celem niniejszego tekstu nie jest dokładne omówienie frameworku, jak równiez opisywanie jego wszystkich możliwości, po więcej informacji zapraszam na strone internetową:
https://sbabic.github.io/swupdate/swupdate.html

#### Pojedyńczy duży obraz
Główną koncepcją projektu SWUpdate jest użycie do aktualizacji pojedyńczego dużego obrazu.
![image-title](/assets/images/SWUpdate_image.png )

Poszczególne pojedyńcze obrazy są spakowane (z pomocą cpio) razem z dodoatkowym plikiem (sw-description). Plik ten opisuje wszystlie komponenty zawarte w obrazie. Więcej na temat pliku będziemy mogli dowiedzieć się przechodząc w przykładach.

## Strategie aktualizacji oprogramowania
### Aktualizacja pojedyńczej aplikacji
Polega na zaktualizowaniu pojedyńczej aplikacji. Najprostszym sposobem przeprowadzenia tego typu aktualizacji jest użycie polecenia scp i podmianę pojedyńczego pliku na dysku. Na początku chciałbym zacząć od zalet takiego rozwiązania. Przede wszystkim jest to mały rozmiar aktualizacji. Drugą istotną zaletą jest to, że zawzwyczaj po aktualizacji wystarczy ponownie uruchomić daną aplikacje a nie cały system operacyjny, aby zmiany zostały zauważone.
Po zaktualizowaniu jednego pliku np. za pomocą polecenia scp mamy w gruncie do czynienia z nową wersją całego oprogramowania. Nikt nie powiedział, że nowa wersja aplikacji będzie działać tak samo z całym systemem jak wersja poprzednia, o ile w ogóle będzie działać. Tego typu podejście może prowadzić szybko do problemów z zależnościami i niespoójności oprogramowania. Kolejnym dużym ryzykiem jest 'popsucie' urządzenia. Jeżeli podczas aktualizacji np. zabraknie Nam prądu Nasze urządzenie może znaleźć się w bliżej nieokreślonym stanie i możemy starcić możliwość ponownej aktualizacji i naprawienia sytacji, przynajmniej zdalnie.

Pomimo dość dużych wad tego pedejścia do aktualizowania oprogramowania jest ono używane dość często.Jest to najszybszy sposób i w przypadku projektów we wczesnej fazie rozwoju, gdzie niezawodnośc nie ogrywa kluczowej roli a urządzenie często mamy pod ręką, podejście sprawdza się idealnie. 

### Przykład
W przykładzie chciałbym pokazać aktualizacje pojedyńczego pliku/aplikacji na płytce Raspberry Pi z pomocą frameworku SWUpdate.
Na potrzeby przykładu zostało użyte narzędzie SWUpdate z włączonym wsparciem sprawdzania sygnatury paczki. Dla ułatwienia można skompilować wersję narzędzia bez sprawdzenia sygnatury. 

#### Wymagania
- Raspberry Pi Board
- Komputer z systemem Linux (ja używałem, Ubuntu 20.04. Ostatecznie można użyć do wszystkiego płytki Raspberry Pi)
- Odrobina wiedzy technicznej i chwila wolnego czasu..


#### Przygotowanie  środowiska
- przygotowanie obrazu systemu Raspian (w przykładzie został użyty Raspberry Pi OS Lite 64-bit). Ważne, aby serwis ssh był domyślnie włączony.
- zapisanie obrazu na karcie SD i zamontowanie jej w płytce
- uruchomienie płytki, podłączenie się po ssh i zainstalowanie SWUpdate z użyciem komendy apt bądź z źródeł. W przykładzie druga opcja została wybrana
{% highlight ruby %}
#install from sources
cd /home/pi
sudo apt-get install lua5.2-dev libssl-dev libconfig-dev libarchive-dev libzmq3-dev libz-dev libcurl4-gnutls-dev libjson-c-dev
#TODO Is this branch needed, error with duplicated case
git clone https://github.com/sbabic/swupdate.git -b 2017.11 && cd swupdate
make test_defconfig
make && sudo make install

#install with apt
sudo apt-get install swupdate
{% endhighlight %}

- przygotowanie pary kluczy (jeśli włączone wsparcie dla podpisywanych paczek) (on host)
{% highlight ruby %}
openssl genrsa -out swupdate-priv.pem
openssl rsa -in swupdate-priv.pem -out swupdate-public.pem -outform PEM -pubout
{% endhighlight %}

- skopiowanie klucza publicznego na płytkę
{% highlight ruby %}
scp swupdate-public.pem  pi@raspberrypi.local:/home/pi/data/security/
{% endhighlight %}

- uruchomienie SWUpdate (on embedded device)
{% highlight ruby %}
sudo swupdate -v -k /home/pi/data/security/swupdate-public.pem -w "-document_root /home/pi/swupdate/www"
{% endhighlight %}

Teraz możemy uruchomić przeglądarkę i uruchomić stronę interfejsu webowego do SWUpdate:
http://raspberrypi.local:8080/: 

OBRAZ OBRAZ
....

#### Aktualizacja pojedyńczego pliku, kroki do zrobienia
 
- stworzenie pliku sw-description (host)
{% highlight ruby %}
software =
{
    version = "1.0.1";
    raspberrypi = {
        hardware-compatibility: [ "1.0" ];
        files: (
            {
                filename = "SWUpdate";
                path = "/tmp/SWUpdate";
                sha256 = "27ebafbe58603437d819cde3470811e5e64a8aa876a4fa680e5266e36bbee519";
            }
        );
    };
}
{% endhighlight %}

- stworzenie pliku aktualizacji (host)
{% highlight ruby %}
echo "SWUpdate v1" > SWUpdate
√ repos % cat ~/workspace/SWUpdate 
SWUpdate v1
{% endhighlight %}

- podpisanie paczki (host)
{% highlight ruby %}
openssl dgst -sha256 -sign ~/swupdate-priv.pem sw-description > sw-description.sig
{% endhighlight %}

- Stworzenie archiwum cpio (host)
{% highlight ruby %}
export FILES="sw-description sw-description.sig SWUpdate"

for i in $FILES; do echo $i; done | cpio -ov -H odc >  update-image-v1.swu
sw-description
SWUpdate
2 blocks
{% endhighlight %}

- Zaktualizowanie płytki z pomocą interfejsu webowego. Wybierz plik: update-image-v1.swu
![image-title](/assets/images/SWUpdate_success.png)

- VWeryfikacja ręczna czy plik został zatualizowany
{% highlight ruby %}
pi@raspberrypi:~ $ sudo cat /tmp/SWUpdate 
SWUpdate v1
{% endhighlight %}

Jak można zauważyć plik został zaktualizowany pomyślnie. W ramach ćwiczeń proponuje wysłać paczkę aktualizacyjną bez lub z niepoprawną sygnaturą i zobaczyć jaki dostaniemy komunikat.
 
Hardware compatibility checking

## Update whole OS image
OS image for embedded device can be often considered as single applications whose individual parts cannot be changed. As example you can image the situation when you provided image for embedded device and later with OS can be updated with help of apt-get command. After updates performed this way OS is different than initial delivered image. Developer can not guarantee that this new image will be work properly becuase it was never tested in that configuration. Maybe after update of some application whole device will not work as expected. Searching of bugs in this kind of customized original image can be nightmare because we never know with which configuration we work.      
Because that reasons the common way for embedded devices is to do update whole image instead of particular files/applications. Main disadvantage of those solution is update size. Whole image is normally bigger than one file/app.

We can distinguish the following types of OS image updates:
- single copy
- dual copy
- diffrential way

In this tutorial i would like to focus on dual copy mode only.

### Dual copy update
In dual copy update the device partitioning look like this:

| BOOTLOADER | OS (active) | OS (inactive) | PERSISTENT DATA |

Can be noticed that we have two OS. One is active and the second one is inactive.
Dual copy update in shortcut contains following steps:
- checking for new updates
- download update to inactive OS
- reboot device and mark inactive OS (updated) as active OS

Now we will pass through simple example of dual copy mode.

git clone git://git.yoctoproject.org/poky -b rocko
git clone https://github.com/openembedded/meta-openembedded.git -b rocko
git clone https://github.com/sbabic/meta-swupdate -b rocko
git clone https://github.com/sbabic/meta-swupdate-boards.git -b master
git clone https://github.com/agherzan/meta-raspberrypi.git -b rocko

greadlink -f layers/poky/oe-init-build-env build
brew install coreutils




