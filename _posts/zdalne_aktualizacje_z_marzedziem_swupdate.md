---
layout: post
title:  "SWUpdate, czyli o aktualizacjach oprogramowania w świecie Embedded Linux"
date:   2023-01-22 12:00:00 +0100
categories: jekyll update
---

## Kilka słów na początek...
W atrykule chciałbym poruszyć temat zdalnych aktualizacji urządzeń Linux Embedded. Przedstawione w nim zostanie narzędzie SWUpdate oraz przy okazji omówione zostaną podstawowe startegie aktualizacji. Pokazany zostanie również przyklad prostej aktualizacji na płytce raspberry pi 4 z wykorzystaniem SWUpdate.

Czasem z rożnych przyczyn urządzenie Linux Embedded nie ma połączenia z siecią Internet. W takim wypadku nie da się przeprowadzić aktualizacji zdalnie tylko trzeba użyć do tego lokalnego interfejsu (USB, SD). W artykule skupie sie wyłącznie na zdalnych aktualizacjach, aczkowliek w przypaku większości poruszonych rozważań medium transmisyjne aktualizacji nie ma większego znaczenia.

Motywacją do napisania artykułu były dla mnie dotychczasowe doświadczenia związane z aktualizacją softu na urządzeniach embedded, które często polegały na wykonaniu 'skryptów' napisanych wcześniej przez programistów. Czasem łatwiej nie tworzyć koła na nowo i można użyć gotowych, dobrze udokumentowanych i przetestowanych rozwiązań. Jest to artykuł raczej dla osób początkujących i ma głównie na celu pokazać czytelnikowi, że istnieje alternatywa dla skryptów oraz przybliżyć narzędzie SWupdate.

## Kilka słów o projekcie SWUpdate
SWUpdate jest to 'Linux Update agent', który za główne zadanie ma Nam pomóc w przeprowadzeniu aktualizacji systemu Embedded Linux. Może zostać zainstalowany za pomocą komendy 'apt' bądz zbudowany ze źródeł. Projekt daje bardzo dużo różnych możliwości i dzięki niemu nie musimy wymyślać wsztkiego od nowa a możemy skorzystać z gotowego, dobrze przetestowanego narzędzia.

Główne funkcjonalności:
- obsługa wielu nośników danych za pomocą różnych handlerów (eMMC, SD, Raw NAND, NOR and SPI-NOR flashes)
- wsparcie wielu interfejsów do pobrania aktualizacji (USB, SD, UART ..)
- możliwość dodawania własnych handlerów
- elastyczny format pliku aktualizacji
- wsparcie aktualizacji wielu komponentów za pomocą pojedyńczego pliku obrazu (swu)
- wsparcie dla zdalnych aktualizacji (OTA support)
- zintegrowany web serwer
- integracja z Hawkbit
- obsługa zaszyfrowanych/podpisanych aktualizacji
- i wiele wiele innych

Celem niniejszego tekstu nie jest dokładne omówienie frameworku, jak równiez opisywanie jego wszystkich możliwości, po więcej informacji zapraszam na strone internetową projektu:
[link](https://sbabic.github.io/swupdate/swupdate.html)

#### Pojedyńczy duży obraz
Główną koncepcją projektu SWUpdate jest użycie do aktualizacji pojedyńczego dużego obrazu.
![image-title](/assets/images/SWUpdate_image.png )

Poszczególne pojedyńcze obrazy są spakowane (z pomocą cpio) razem z dodatkowym plikiem (sw-description). Plik ten opisuje wszystlie komponenty zawarte w obrazie. Więcej na temat pliku będziemy mogli dowiedzieć się w przykładach.

## Strategie aktualizacji oprogramowania
### Aktualizacja pojedyńczej aplikacji
Polega na zaktualizowaniu pojedyńczej aplikacji. Najprostszym sposobem przeprowadzenia tego typu aktualizacji jest użycie polecenia scp i podmianę pojedyńczego pliku na dysku.

Na początku chciałbym zacząć od zalet takiego rozwiązania. Przede wszystkim jest to mały rozmiar aktualizacji. Drugą istotną zaletą jest to, że zazwyczaj po aktualizacji wystarczy ponownie uruchomić daną aplikacje a nie cały system operacyjny, aby zmiany zostały zauważone.

Po zaktualizowaniu jednego pliku np. za pomocą polecenia scp mamy w gruncie rzeczy do czynienia z nową wersją całego oprogramowania. Nikt nie powiedział, że nowa wersja aplikacji będzie działać tak samo z całym systemem jak wersja poprzednia, o ile w ogóle będzie działać. Tego typu podejście może prowadzić szybko do problemów z zależnościami i niespoójności oprogramowania. Kolejnym dużym ryzykiem jest 'popsucie' urządzenia. Jeżeli podczas aktualizacji np. zabraknie Nam prądu Nasze urządzenie może znaleźć się w bliżej nieokreślonym stanie i możemy starcić możliwość ponownej aktualizacji i naprawienia sytuacji, przynajmniej zdalnie.

Pomimo sporych wad tego podejścia do aktualizowania oprogramowania jest ono używane dość często. Jest to najszybszy sposób i w przypadku projektów we wczesnej fazie rozwoju, gdzie niezawodnośc nie ogrywa kluczowej roli a urządzenie często mamy pod ręką, podejście sprawdza się idealnie. 

#### **Przykład 1 - aktualizacje pojedyńczego pliku**
Na potrzeby przykładu zostało użyte narzędzie SWUpdate z włączonym wsparciem sprawdzania sygnatury paczki. Dla ułatwienia można skompilować wersję narzędzia bez sprawdzenia sygnatury. 

#### Wymagania
- Raspberry Pi Board
- komputer z systemem Linux (ja używałem Ubuntu 20.04, ostatecznie można użyć do wszystkiego płytki Raspberry Pi)
- odrobina wiedzy technicznej i chwila wolnego czasu


#### Przygotowanie środowiska
- przygotowanie obrazu systemu Raspian (w przykładzie został użyty Raspberry Pi OS Lite 64-bit). Ważne, aby serwis ssh był domyślnie włączony.
- zapisanie obrazu na karcie SD i zamontowanie jej w płytce
- uruchomienie płytki, podłączenie się po ssh i zainstalowanie SWUpdate z użyciem komendy apt bądź ze źródeł. W przykładzie została wybrana druga opcja.
{% highlight ruby %}
#install from sources
cd /home/pi
sudo apt-get install lua5.2-dev libssl-dev libconfig-dev libarchive-dev libzmq3-dev libz-dev libcurl4-gnutls-dev libjson-c-dev
git clone https://github.com/sbabic/swupdate.git && cd swupdate
make test_defconfig
// Optionally: make menuconfig
make && sudo make install

#install with apt
sudo apt-get install swupdate
{% endhighlight %}

- przygotowanie pary kluczy (jeśli zostało włączone wsparcie dla podpisywanych paczek) (host)
{% highlight ruby %}
openssl genrsa -out swupdate-priv.pem
openssl rsa -in swupdate-priv.pem -out swupdate-public.pem -outform PEM -pubout
{% endhighlight %}

- skopiowanie klucza publicznego na płytkę (host)
{% highlight ruby %}
scp swupdate-public.pem  pi@raspberrypi.local:/home/pi/data/security/
{% endhighlight %}

- uruchomienie SWUpdate (on embedded device)
{% highlight ruby %}
sudo swupdate -v -k /home/pi/data/security/swupdate-public.pem -w "-document_root /home/pi/swupdate/www"
{% endhighlight %}

Teraz możemy uruchomić przeglądarkę i uruchomić stronę interfejsu webowego do SWUpdate:
http://raspberrypi.local:8080

![image-title](/assets/images/SWUpdate_start.png)

#### Aktualizacja, kroki do zrobienia
 
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
cat ~/workspace/SWUpdate 
SWUpdate v1
{% endhighlight %}

- podpisanie paczki (host)
{% highlight ruby %}
openssl dgst -sha256 -sign ~/swupdate-priv.pem sw-description > sw-description.sig
{% endhighlight %}

- stworzenie archiwum cpio (host)
{% highlight ruby %}
export FILES="sw-description sw-description.sig SWUpdate"

for i in $FILES; do echo $i; done | cpio -ov -H odc >  update-image-v1.swu
sw-description
SWUpdate
2 blocks
{% endhighlight %}

- zaktualizowanie płytki z pomocą interfejsu webowego. Wybierz plik: update-image-v1.swu
![image-title](/assets/images/SWUpdate_success.png)

- weryfikacja ręczna czy plik został zatualizowany (rp4)
{% highlight ruby %}
pi@raspberrypi:~ $ sudo cat /tmp/SWUpdate 
SWUpdate v1
{% endhighlight %}

Jak można zauważyć plik został zaktualizowany pomyślnie. W ramach ćwiczeń proponuje wysłać paczkę aktualizacyjną bez lub z niepoprawną sygnaturą i zobaczyć jaki dostaniemy komunikat.
 
#### **Przyklad 2 - hardware compatibility**
Jedną z opcji jakie umożliwia Nam framework SWUpdate jest sprawdzenie kompatybilności aktualizacji ze sprzętem na którym ma zostać zainstalowana. Na urządzeniu Linux Embedded musimy stworzyć plik /etc/hwrevision (sciezka może być inna, ale wtedy trzeba powiedzieć o tym SWUpdate podczas uruchamiania poprzez parametr). Plik ten opisuje rewizje hardwaru jaki posiadamy. W moim przypadku plik wygląda następująco:
{% highlight ruby %}
pi@raspberrypi:~ $ cat /etc/hwrevision 
raspberrypi 1.0
{% endhighlight %}

Kolejnym krokiem jest dodanie w pliku sw-description rewizji hardwaru, dla których przeznaczona jest aktualizacja:
{% highlight ruby %}
software =
{
    version = "1.0.1";
    raspberrypi = {
        hardware-compatibility: [ "1.0" ];
        ....
{% endhighlight %}

W przypadku kiedy aktualizacja nie będzie pasować do rewizji sprzętu, nie zostanie ona zainstalowana a my zostaniemy o tym poinformowani co poszło nie tak:
![image-title](/assets/images/SWUpdate_hwrevision.png)


## Aktualizacje pełnego obrazu systemu
Kolejną strategią aktualizacji jest aktualizacja nie pojedyńczego pliku czy aplikacji, a całego systemu. Dostarczając pełen obraz systemu nie musimy się martwić spójnością oprogramowania ponieważ dostarczamy oprogramowanie w danej konfiguracji ktora została przez Nas wcześniej przetestowana. Kolejna zaletą jest to, że jeśli Nasza aktualizacja się nie uda, albo co gorsza zabraknie zasilania w samym środku aktualizacji mamy dostępne mechanizmy, ktore pomogą nam wyjść z opresji. Jej największą wadą jest rozmiar przesyłanej paczki. Może być to szczególnie problemem kiedy mamy ograniczone możliwości w zakresie transferu danych. To podejście wymaga od Nas trochę więcej wiedzy (bootloader itp), pracy oraz czasu.

Możemy wyrożnić tutaj trzy różne podejścia:
- single copy
- dual copy
- differential update

### Dual copy
Partycjonowanie urządzenia wygląda następująco:

| Bootloader | Podstawowy OS (aktywny) | Podstawowy OS (nieaktywny) | Partycja danych |

Można zauważyć, że w przypadku strategii Dual Copy mamy dwie partycje zawierające system operacyjny. W danej chwili aktywny jest tylko jeden OS, drugi pozostaje niekatywny.

Zasada działania aktualizacji jest bardzo prosta. Urządzenie odpytuje się o dostępne aktualizacje. Kiedy otrzyma pozytywną odpowiedź to pobiera nowy obraz i zapisuje go na niekatywnej partycji. W kolejnym kroku urządzenie musi zostać uruchomione ponownie z nową zapisaną partycją oznaczoną jako aktywna.

Tym sposobem doszliśmy do zalet tego podejścia. W przypadku kiedy nowo zapisany obraz systemu nie uruchamia się poprawnie zawsze możemy powrocić do poprzedniej wersji i zrobić ją aktywną. Kiedy podczas aktualizacji zabraknie nam zasilania urządzenie nadal będzie mogło korzystać ze 'starego' obrazu systemu i ponowić próbę aktualizacji. Plusem jest rownież to, że nie potrzebujemy żadnej interakcji z użytkownikiem ponieważ wszystkie operacje związane z aktualizacją mogą zostać wykonane w tle.

Wadą rozwiązania jest ilość potrzebnego miejsca. Można powiedzieć, że potrzebujemy minimum dwa razy tyle miejsca na dysku jak w przypadku aktualizacji pojedynczej aplikacji na urządzeniu z jednym OS. Pewną niedogodnoscią może być rownież to, ze wymagane jest ponowne uruchomienie urządzenia.

### Single Copy
Partcjonowanie urządzenia wygląda nastepujaco:

| Bootloader | Podstawowy OS (aktywny)| Recovery OS| Partycja danych |

Podstawową rożnicą między startegią single copy a dual copy jest to, że w przypadku tej pierwszej używamy Recovery OS zamiast drugiej partycji z pełnym OS. Recovery OS jest to minimalny obraz systemu z którego można wystartować urządzenie oraz dostarcza możliwość zainstalowania aktualizacji.

Aktualizacja można podzielić na następujace kroki. Urządzenie odpytuje się o dostępne aktualizacje i jeśli takową znajdzie pobiera obraz i zapisuje go na partycji danych. Następnie prosi użytkownika o rozpoczęcie instalowania aktualizacji. W kolejnym kroku urządzenie zostaje uruchomione z Recovery OS, a pobrany wcześniej obraz nadpisuje Regular OS. Na koniec pozostaje uruchomić urządzenie ładujac podstawowy obraz systemu.

Jest niestety kilka istotnych wad tego podejscia. Potrzebujemy sporo miejsca na partycji danych ponieważ obraz musi zostać gdzieś pobrany. Urządzenie przez czas instalowania aktualizacji pozostaje niaktywne. W razie nieudanej aktualizacji pozostajemy z urządzeniem w trybie recovery, ale z dobrych stron możemy spróbowac zainstalowac aktualizacje jeszcze raz. Niestety nie mozemy pobrać obrazu jeszcze raz bo ten krok jest wykonanywany przy użyciu podstawowego OS a nie recovery. W takim wypadku jeśli wcześniej pobrany obraz jest w jakiś sposob uszkodzony, bedziemy musieli podjąć się manualnej naprawy problemu.

### Differential update
Jest to rodzaj aktualizacji który rozwiązuje problem z jej wielkością. Polega no pobieraniu i zapisywaniu różnic między starym a nowym obrazem. Nie będę sie tutaj więcej rozpisywał ponieważ jest to dość obszerny temat. Po więcej informacji zapraszam na stronę: [link](https://sbabic.github.io/swupdate/delta-update.html)

## Podsumowanie
Framework SWUpdate pozwala Nam podejść do aktualizacji w prosty i usystemtyzowany sposób. Dostarcza on Nam wiele ciekawych możliwości o których byłoby warto jeszcze wspomnieć. Szczerze mówiąc, zagadnienia poruszone w artykule są tylko małym skrawkiem wszystkich możliwości frameworku. Warto byłoby wspomnieć, najlepiej na przykładzie, o aktualizacji z wykorzystaniem pełnego obrazu systemu. Kolejnym ciekawym tematem jest integracja z serwerem HawkBit (serwer aktualizacji). Integracja ta pozwala Nam stworzyć kompleksowy system do zarządzania aktualizacjami. Podsumowując, zostaje jeszcze dużo ciekawych opcji którym można by się przyjrzeć bardziej szczegółowo, ale są to już tematy na kolejne artykuły.
