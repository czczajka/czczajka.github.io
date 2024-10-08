# SWUpdate, czyli o aktualizacjach oprogramowania w świecie Embedded Linux

## Kilka słów na początek...
W artykule chciałbym poruszyć temat zdalnych aktualizacji urządzeń Linux Embedded. Omówione zostaną podstawowe strategie aktualizacji oprogramowania oraz przedstawione zostanie narzędzie SWUpdate. W artykule znajdziemy przykłady prostych aktualizacji z wykorzystaniem płytki Raspberry Pi 4 oraz frameworku SWUpdate. Motywacją do napisania artykułu były dla mnie dotychczasowe doświadczenia związane z aktualizacjami urządzeń embedded, które często sprowadzały się do wykonaniu skryptów napisanych wcześniej przez programistów. Jako alternatywę do tego podejścia możemy użyć gotowego, dobrze przetestowanego narzędzia SWUpdate.

Czasem z rożnych przyczyn urządzenie Linux Embedded nie ma połączenia z siecią Internet. W takim wypadku nie da się przeprowadzić aktualizacji zdalnie tylko trzeba użyć do tego lokalnego interfejsu (UART, SD). W przykładach skupię się na zdalnych aktualizacjach, aczkolwiek w przypadku większości poruszonych rozważań medium transmisyjne wykorzystane do aktualizacji nie ma większego znaczenia.

## Strategie aktualizacji oprogramowania
### Aktualizacja pojedynczej aplikacji
Polega na zaktualizowaniu pojedynczej aplikacji. Najprostszym sposobem przeprowadzenia tego typu aktualizacji jest np. użycie polecenia `scp` i podmianę pliku/plików aplikacji na dysku.

Na początku chciałbym zacząć od zalet takiego rozwiązania. Pierwszą korzyścią jest mniejszy rozmiar aktualizacji w porównaniu do aktualizacji pełnego obrazu systemu. Drugą jest to, że zazwyczaj po aktualizacji wystarczy ponownie uruchomić daną aplikacje a nie cały system operacyjny, aby zmiany zostały zauważone.

Po zaktualizowaniu pojedynczej aplikacji mamy w gruncie rzeczy do czynienia z nową wersją całego oprogramowania. Nowa wersja aplikacji może nie działać poprawnie z całym systemem jak wersja poprzednia. Podejście to może prowadzić szybko do problemów z zależnościami i niespójnością oprogramowania. Kolejnym ryzykiem jest "popsucie" urządzenia. Jeżeli podczas aktualizacji np. zabraknie zasilania urządzenie może znaleźć się w bliżej nieokreślonym stanie i możemy stracić możliwość ponownej aktualizacji i naprawienia sytuacji, przynajmniej zdalnie.

Strategia aktualizacji pojedynczej aplikacji jest jednak często wystarczająca i przede wszystkim szybka oraz prosta w użyciu. W przypadku projektów we wczesnej fazie rozwoju, gdzie niezawodność nie ogrywa kluczowej roli a urządzenie często mamy pod ręką, podejście to bardzo dobrze się sprawdza. 

### Aktualizacje pełnego obrazu systemu
Kolejną strategią aktualizacji jest aktualizacja nie pojedynczej aplikacji, a całego systemu. Dostarczając pełen obraz systemu nie musimy się martwić spójnością oprogramowania, ponieważ dostarczamy oprogramowanie w danej konfiguracji, która została wcześniej przetestowana. Kolejną zaletą jest to, że jeśli aktualizacja się nie uda, albo zabraknie zasilania podczas aktualizacji są dostępne mechanizmy, które pomogą nam wyjść z opresji. Jej wadą jest rozmiar przesyłanej paczki. Może być to szczególnie problemem kiedy transfer danych jest ograniczony. To podejście wymaga też więcej wiedzy (bootloader itp).

Możemy wyróżnić tutaj trzy różne strategie:
- single copy,
- dual copy,
- differential update.

#### Dual copy
Partycjonowanie urządzenia wygląda następująco:

| Bootloader | Podstawowy OS (aktywny) | Podstawowy OS (nieaktywny) | Partycja danych |

W przypadku strategii dual copy mamy dwie partycje zawierające system operacyjny. W danej chwili aktywny jest tylko jeden OS, drugi pozostaje nieaktywny.

Zasada działania aktualizacji jest prosta. Urządzenie odpytuje się o dostępne aktualizacje. Kiedy otrzyma pozytywną odpowiedź to pobiera nowy obraz i zapisuje go na nieaktywnej partycji. W kolejnym kroku urządzenie musi zostać uruchomione ponownie z nową zapisaną partycją oznaczoną jako aktywna.

Tym sposobem doszliśmy do zalet tego podejścia. W przypadku, kiedy nowo zapisany obraz systemu nie uruchamia się poprawnie zawsze można powrócić do poprzedniej wersji i zrobić ją aktywną. Kiedy podczas aktualizacji zabraknie zasilania urządzenie nadal będzie mogło korzystać ze "starego" obrazu systemu i ponowić próbę aktualizacji. Plusem jest również to, że nie potrzebujemy żadnej interakcji z użytkownikiem, ponieważ wszystkie operacje związane z aktualizacją mogą zostać wykonane w tle.

Wadą rozwiązania jest ilość potrzebnego miejsca. Można powiedzieć, że potrzebujemy minimum dwa razy tyle miejsca na dysku jak w przypadku aktualizacji pojedynczej aplikacji na urządzeniu z jednym OS. Pewną niedogodnością może być również to, że wymagane jest ponowne uruchomienie urządzenia.

#### Single Copy
Partcjonowanie urządzenia wygląda następująco:

| Bootloader | Podstawowy OS (aktywny)| Recovery OS| Partycja danych |

Podstawową różnicą między strategią single copy a dual copy jest to, że w przypadku tej pierwszej używamy Recovery OS zamiast drugiej partycji z pełnym OS. Recovery OS jest to minimalny obraz systemu, z którego można wystartować urządzenie oraz dostarcza możliwość zainstalowania aktualizacji.

Aktualizacje można podzielić na następujące kroki. Urządzenie odpytuje się o dostępne aktualizacje i jeśli takową znajdzie pobiera obraz i zapisuje go na partycji danych. Następnie prosi użytkownika o rozpoczęcie instalowania aktualizacji. W kolejnym kroku urządzenie zostaje uruchomione z Recovery OS, a pobrany wcześniej obraz nadpisuje Regular OS. Na koniec pozostaje uruchomić urządzenie ładując podstawowy obraz systemu.

Jest niestety kilka wad tego podejścia. Potrzebujemy sporo miejsca na partycji danych ponieważ obraz musi zostać gdzieś pobrany. Urządzenie przez czas instalowania aktualizacji pozostaje nieaktywne. W razie nieudanej aktualizacji pozostajemy z urządzeniem w trybie recovery, ale z dobrych stron możemy spróbowac zainstalować aktualizacje jeszcze raz. Niestety nie możemy pobrać obrazu jeszcze raz bo ten krok jest wykonanywany przy użyciu podstawowego OS a nie recovery. W takim wypadku jeśli wcześniej pobrany obraz jest w jakiś sposób uszkodzony, będziemy musieli podjąć się manualnej naprawy problemu.

#### Differential update
Jest to rodzaj aktualizacji który rozwiązuje problem z jej wielkością. Polega na pobieraniu i zapisywaniu różnic między starym a nowym obrazem. Nie będe się tutaj więcej rozpisywał ponieważ jest to obszerny temat. Po więcej informacji zapraszam na stronę: [link](https://sbabic.github.io/swupdate/delta-update.html).

## Czym jest SWUpdate
SWUpdate jest to 'Linux Update agent', który za główne zadanie ma pomóc w przeprowadzeniu aktualizacji systemu Embedded Linux. Może zostać zainstalowany za pomocą komendy menadżera pakietów bądź zbudowany ze źródeł. Projekt oferuje dużo możliwości i dzięki niemu nie musimy wymyślać wszystkiego od nowa.

Główne funkcjonalności:
- wsparcie wielu strategii aktualizacji, pojedynczych aplikacji oraz całych partycji,
- obsługa wielu nośników danych za pomocą różnych handlerów (eMMC, SD, Raw NAND, NOR, ...),
- obsługa wielu lokalnych interfejsów do pobrania oprogramowania (USB, SD, SPI, UART),
- możliwość przeprowadzania zdalnych aktualizacji,
- zintegrowany web serwer do przeprowadzenia aktualizacji,
- tryb ciągłego odpytywania się o dostępne aktualizacje,
- możliwa integracja z serwerami do zarządzania aktualizacjami. Obecna wersja wspiera serwer Hawkbit,
- obsługa zaszyfrowanych/podpisanych aktualizacji,
- sprawdzanie kompatybilności pomiędzy rewizją oprogramowania i urządzenia,
- możliwość dodawania własnych handlerów do obsługi niestandardowych protokołów.

Wymienione została tylko część możliwości jakie oferuje framework SWUpdate. Po więcej informacji zapraszam na stronę internetową projektu: [link](https://sbabic.github.io/swupdate/swupdate.html)

### Budowa pliku aktualizacji
Jednym z założeń projektu SWUpdate jest użycie do aktualizacji pojedynczego pliku.
<p align="center">
  <img src="/assets/images/swupdate/swupdate.drawio.svg" />
</p>

- sw-description - zawiera meta informacje o pliku aktualizacji. Plik aktualizacji może zawierać wiele obrazów/plików. SWUpdate używa biblioteki „libconfig” jako domyślnego parsera pliku sw-description. Istnieje jednak możliwość rozszerzenia SWUpdate i dodania własnego parsera, opartego na innej składni i języku niż ten obsługiwany przez libconfig,
- image 1 ... image n - poszczególne obrazy/pliki użyte podczas aktualizacji.

Wszystkie pojedyncze obrazy/pliki są spakowane razem wraz z plikiem sw-description jako archiwum `cpio` (wybrano `cpio` ze względu na jego prostotę i możliwość przesyłania strumieniowego).

## Przykłady
### Wymagania
Do wykonania przykładów potrzebujemy urządzenia z systemem Linux oraz komputera, z którego przeprowadzimy aktualizacje urządzenia. W moim przypadku użyłem płytki Raspberry Pi 4 oraz komputera z zainstalowanym Ubuntu 22.04.

### Przygotowanie środowiska
1. Przygotowanie obrazu systemu Raspian. Ważne, aby serwis `ssh` był domyślnie włączony. [host]
2. Zapisanie obrazu na karcie SD i zamontowanie jej w płytce. [host, Raspberry Pi]
3. Zainstalowanie narzędzia SWUpdate na urządzeniu, którego oprogramowanie będzie aktualizowane. SWUpdate można dostarczyć z użyciem menadżera pakietów bądź zbudować ze źródeł. W przykładzie została wybrana druga opcja. [Raspberry Pi]   
    ```
    #install with apt
    sudo apt-get install swupdate

    #install from sources
    cd /home/pi
    sudo apt-get install lua5.2-dev libssl-dev libconfig-dev libarchive-dev libzmq3-dev libz-dev libcurl4-gnutls-dev libjson-c-dev
    git clone https://github.com/sbabic/swupdate.git && cd swupdate
    make test_defconfig
    // Optionally: make menuconfig
    make && sudo make install
    ```
4. Przygotowanie pary kluczy (jeśli wcześniej zostało włączone wsparcie dla podpisywanych paczek). [host]
    ```
    openssl genrsa -out swupdate-priv.pem
    openssl rsa -in swupdate-priv.pem -out swupdate-public.pem -outform PEM -pubout
    ```
5. Skopiowanie klucza publicznego na płytkę. [host]
    ```
    scp swupdate-public.pem pi@raspberrypi.local:/home/pi/data/security/
    ```
6. Uruchomienie SWUpdate. [Raspbery Pi]
    ```
    sudo swupdate -v -k /home/pi/data/security/swupdate-public.pem -w "-document_root /home/pi/swupdate/www"
    // -v - ustawia maksymalny poziom logowania
    // -k - wskazuje na lokalizacje klucza publicznego, który zostanie użyty do weryfikacji instalowanej aktualizacji
    // -w - mówi, że zostanie uruchomiony wbudowany webserwer
    // -document_root - ścieżka gdzie znajduje się `document root` webservera. Jeśli SWUpdate został zainstalowany za pomocą menadżera pakietów to trzeba pobrać repozytorium projektu, ponieważ w nim znajduje się wymagany "document root" dla webservera 
    ```
7. Jako ostatni krok w celu weryfikacji możemy uruchomić przeglądarkę i wprowadzić stronę interfejsu webowego SWUpdate: http://IP-URZĄDZENIA:8080. [host]
    ![image-title](/assets/images/swupdate/SWUpdate_start.png)

### Przykład 1 - aktualizacje pojedynczej aplikacji
#### Aktualizacja, krok po kroku
1. Stworzenie pliku sw-description. [host]
    ```
    software =
    {
        version = "1.0.1";
        raspberrypi = {
            hardware-compatibility: [ "1.0" ];
            files: (
                {
                    filename = "myApp";
                    path = "/usr/bin/myApp";
                    sha256 = "a25c88c564ad0e8c42d9863c3a4309c88a264a4b2fca56031e835f05a8e653d1";
                }
            );
        };
    }
    ```
Opis parametrów w sw-description:
version: wersja do której się aktualizujemy,
filename: nazwa aplikacji, która zostanie zaktualizowana,
path: ścieżka dostępu do aktualizowanej aplikacji,
sha256: hash nowej wersji aplikacji.

2. Podpisanie paczki. [host]
    ```
    openssl dgst -sha256 -sign ~/swupdate-priv.pem sw-description > sw-description.sig
    ```
3. Stworzenie archiwum cpio, w moim przypadku robię to w Bashu. [host]
    ```
    FILES="sw-description sw-description.sig myApp"

    for i in $FILES; do echo $i; done | cpio -ov -H odc >  update-image-v1.swu
    sw-description
    sw-description.sig
    myApp
    3 blocks
    ```
4. Zaktualizowanie płytki z pomocą interfejsu webowego. Krok ten wymaga uruchomienia przeglądarki oraz uruchomienia interfejsu webowego SWUpdate. Adres mojej płytki Raspberry Pi 4 jest następujący: http://raspberrypi.local:8080. Następnie za pomocą przycisku na stronie wskazujemy plik z aktualizacją: update-image-v1.swu [host]
![image-title](/assets/images/swupdate/SWUpdate_success.png)
5. W celu weryfikacji czy aplikacja została zaktualizowana poprawnie podłączam się za pomocą `ssh` do płytki i sprawdzam czy zawartość pliku aplikacji została podmieniona na nową wersje. Wcześniej w pliku myApp znajdował się teks `myApp v0` [Raspbery Pi]
```
pi@raspberrypi:~ $ sudo cat /usr/bin/myApp 
myApp v1
```
Aplikacja została zaktualizowana pomyślnie. W ramach ćwiczeń proponuje wysłać paczkę aktualizacyjną bez lub z niepoprawną sygnaturą i zobaczyć jaki dostaniemy komunikat.
 
### Przykład 2 - hardware compatibility
Jedną z opcji jakie umożliwia framework SWUpdate jest sprawdzenie kompatybilności aktualizacji ze sprzętem na którym ma zostać zainstalowana. 
1. Na urządzeniu Linux Embedded trzeba stworzyć plik /etc/hwrevision, scieżka może być inna, ale wtedy trzeba powiedzieć o tym SWUpdate podczas uruchamiania poprzez parametr. Plik ten opisuje rewizje hardwaru. Na moim urządzeniu plik wygląda następująco:
    ```
    pi@raspberrypi:~ $ cat /etc/hwrevision
    raspberrypi 1.0
    ```
2. Kolejnym krokiem jest dodanie w pliku sw-description rewizji- hardware compatibility, dla których przeznaczona jest aktualizacja:
```
software =
{
    version = "1.0.1";
    raspberrypi = {
        // soft kompatybilny z hardwarem
        hardware-compatibility: [ "1.0" ];
        // niekompatybilny soft z hardwarem
        // hardware-compatibility: [ "3.0" ];
        ...
    }
    ...
}
```
W przypadku kiedy aktualizacja nie będzie kompatybilna z rewizją sprzętu, nie zostanie ona zainstalowana i zwróci komunikat błędu:
![image-title](/assets/images/swupdate/SWUpdate_hwrevision.png)

## Podsumowanie
Framework SWUpdate pozwala podejść do aktualizacji w prosty i bezpieczny sposób. Jest rozbudowanym narzędziem i posiada wiele innych, ciekawych możliwości jak aktualizacje z wykorzystaniem pełnego obrazu systemu bądź integracja z serwerem HawkBit, z którymi polecam się zapoznać.