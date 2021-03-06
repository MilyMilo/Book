# Inteligentny dom z (Micro)Python - Krzysztof Czarnota

Inteligentne domy stają się coraz bardziej powszechne. Nic dziwnego,
rozwiązania, które ułatwiają nam codzienne życie, a także pozwalają
na oszczędność czasu i&nbsp;pieniędzy muszą cieszyć się dużą popularnością.
Na rynku pojawia się wiele firm, które oferują swoje usługi w zakresie
automatyzacji budynków, ale niejednokrotnie koszty takiej instalacji
odstraszają potencjalnego użytkownika. Tymczasem stworzenie systemu
inteligentnego domu wcale nie musi być drogie ani trudne.

## Jak zacząć?
Instalacja inteligentnego domu składa się z sieci czujników oraz elementów
wykonawczych, która to sieć monitoruje oraz kontroluje obecny stan budynku.
Druga część instalacji to centralny system zarządzania, który pozwala
na personalizację ustawień, a następnie, na podstawie informacji z czujników,
steruje urządzeniami wykonawczymi. W budynkach (zwłaszcza w nowych
inwestycjach) instalowanych jest obecnie wiele urządzeń pozwalających
na zdalne sterowanie. Jednak nie ma jeszcze jednolitego standardu, który
umożliwiałby bezproblemowe połączenie i sterowanie wszystkimi urządzeniami.
W naszej instalacji muszą pojawić się także elementy pozwalające na sterowanie
urządzeniami, które oryginalnie nie zapewniają takiej funkcjonalności.
Najprostszym rozwiązaniem obu tych problemów jest wykonanie własnego projektu sprzętowego.

Ale bez obaw, stworzenie urządzenia elektronicznego przestało wiązać się
z niskopoziomowym programowaniem mikrokontrolera i studiowaniem not
aplikacyjnych układów scalonych. Silnie rozwijający się rynek IoT
(Internet of Things) oraz platform prototypowych daje do dyspozycji cały
szereg modułów elektronicznych gotowych do wykorzystania bez wgłębiania się
w ich budowę, a nowoczesne narzędzia deweloperskie pozwalają nam na napisanie
własnego systemu wbudowanego w języku wysokiego poziomu. Niemniej takie
podejście sprawdzi się tylko na początku, dlatego zachęcam do pogłębiania
swojej wiedzy w dziedzinie elektroniki. Interakcja ze sprzętem daje wiele
radości, a uruchomione urządzenie potrafi przynieść wymierne korzyści.

W celu rozpoczęcia pracy niezbędne jest zaopatrzenie się w kilka urządzeń oraz
modułów, które znajdą zastosowanie w każdej instalacji inteligentnego budynku:

* Raspberry Pi - centrum zarządzania budynkiem,
* Moduły ESP-12F - bezprzewodowa komunikacja z czujnikami i urządzeniami wykonawczymi,
* Konwerter USB-UART - komunikacja przez port szeregowy,
* Moduł przekaźnika RM0 - umożliwia włączanie/wyłączanie urządzeń 230V,
* Moduł DS18b20 - termometr 1-wire,
* Zasilacz 5V oraz 3,3V.

Już tak krótka lista pozwoli na sterowanie oświetleniem czy sprawowanie
kontroli nad temperaturą w domu. Na początku, do testowania funkcjonalności
nowych modułów elektronicznych, można wykorzystać jedną z platform
prototypowych np. Arduino.

## MicroPython i moduł ESP8266
MicroPython to implementacja języka Python 3, która zawiera niewielki podzbiór
biblioteki standardowej języka Python i jest zoptymalizowana pod kątem
działania na mikrokontrolerach [1]. Istnieje kilka platform
sprzętowych pozwalających na uruchomienie środowiska MicroPython takich
jak pyBoard czy WiPy. Układem, na którym oprzemy naszą instalację
inteligentnego domu, będzie ESP8266 [2], a konkretnie moduł o nazwie
ESP-12F.
Co oferuje nam urządzenie wielkości znaczka pocztowego (24×16mm) za niecałe
2 dolary? Oto podstawowe cechy modułu ESP-12F:

- 32 bit RISC CPU - Tensilica Xtensa LX106 @80MHz,
- 11 wyprowadzeń GPIO - wejścia/wyjścia cyfrowe,
- 1 wejście ADC - 10 bitowy przetwornik analogowo-cyfrowy,
- interfejsy - SPI, I2C, 2×UART,
- WiFi 802.11 b/g/n (2,4GHz),
- 4MB flash.

Komunikacja z modułem ESP-12F odbywa się za pomocą portu szeregowego.
Konwerter USB-UART należy podłączyć do wyprowadzeń oznaczonych jako TXD oraz
RXD. Trzeba zwrócić uwagę, że komponent zasilany jest napięciem 3,3V i pracuje
na logice o tym samym potencjale. Do zasilania układu należy wykorzystać
źródło napięcia o wydajności około 300mA. W celu uruchomienia modułu wymagane
jest również podciągnięcie pinu EN (`CH_PD`) do VCC oraz pinu GPIO15 do GND
przez rezystory 10kΩ.

Oryginalne oprogramowanie modułu obsługuje komunikację za pomocą komend AT.
W celu uruchomienia środowiska MicroPython należy wgrać nowy firmware.
Najlepszym sposobem uzyskania oprogramowania jest zbudowanie najnowszej wersji
ze źródeł. Szczegółowy opis tego procesu znajduje się w repozytorium z kodem
źródłowym [3], można również wykorzystać znalezione w sieci
gotowe pliki binarne firmware. Kolejnym krokiem jest wyczyszczenie pamięci
układu i wgranie nowego firmware. Do wykonania tych czynności wykorzystamy
narzędzie esptool, które służy do komunikacji z bootloaderem w układzie
ESP8266 [4]. Przed wpisaniem podanej poniżej komendy należy pamiętać
o uruchomieniu modułu ESP w trybie programowania (flash mode), wykonuje się to
przez podłączenie pinu GPIO0 do masy.

    $ esptool.py --port /dev/tty.xxx --baud 460800 write_flash --flash_size=8m 0 firmware-combined.bin

W przypadku powodzenia naszym oczom powinien ukazać się następujący rezultat:

    Connecting...
    Erasing flash...
    Writing at 0x0007cc00... (100 %)

    Leaving...

Moduł jest gotowy do uruchomienia środowiska MicroPython. Po odłączeniu pinu
GPIO0 od masy i zrestartowaniu modułu bootloader załaduje nowy wsad. Możemy
już uzyskać dostęp do REPL (Python prompt) poprzez port szeregowy.

    $ screen /dev/tty.xxx 115200

Czas na pierwszy program. W świecie programowania mikrokontrolerów
odpowiednikiem wypisania tekstu "Hello, world" jest program powodujący miganie
diody LED (chociaż w naszym przypadku nic nie stoi na przeszkodzie wypisania
tekstu). Poniżej znajduje się kod, który sprawia, że niebieska dioda LED
zamontowana na module będzie migać z częstotliwością 1Hz.

    >>> import machine
    >>> import time
    >>> pin = machine.Pin(2, machine.Pin.OUT)
    >>> while True:
    ...     pin.high()
    ...     time.sleep_ms(500)
    ...     pin.low()
    ...     time.sleep_ms(500)

Gotowe! Właśnie zrealizowaliśmy pierwszy projekt sprzętowy. Więcej informacji
pomocnych dla początkujących dostępne jest w tutorialu środowiska MicroPython
dla układu ESP82666 [5].


## Raspberry Pi
Raspberry Pi to komputer wielkości karty kredytowej (85×56mm), który znajduje
zastosowanie w projektach elektronicznych zastępując komputer stacjonarny
[6]. Specyfikacja sprzętowa najnowszej wersji Raspberry Pi&nbsp;3 model&nbsp;B jest następująca:
* 64 bit CPU - Quad-Core ARM Cortex A53 @1,2GHz,
* Pamięć RAM - 1GB LPDDR2 @900MHz,
* Interfejsy - 4×USB 2.0, UART, SPI, I2C, GPIO,
* Sieć - Ethernet 10/100Mbps, Wifi 802.11 b/g/n 150Mbps, Bluetooth Low Energy, BLE 4.1.

Istnieją różne warianty sprzętowe (Model A, Model B, Zero), co pozwala
na dobranie urządzenia do potrzeb projektu. Raspberry Pi doskonale sprawdzi
się jako centrum zarządzania inteligentnym domem. Oficjalnym system
operacyjnym jest Raspbian, jest to dystrybucja linuxa oparta na Debianie
co sprawia, że z łatwością uruchomimy dowolną usługę czy skrypt niezbędny
do obsługi czujników oraz urządzeń w domu. Możliwość podłączenia wyświetlacza
dotykowego o rozdzielczości 800×480 pikseli pozwala na zapewnienie wygodnego
interfejsu użytkownika.

## Komunikacja z urządzeniami wykonawczymi
Trudno wyobrazić sobie system inteligentnego domu bez możliwości kontrolowania
urządzeń zainstalowanych w domu takich jak piec, centrala wentylacyjna
czy instalacja ogrzewania podłogowego. Przeważnie każde z takich urządzeń
posiada dedykowany sterownik, który sprawuje kontrolę nad pracą danego
urządzenia. W przypadku prostych urządzeń sterowanie włącz/wyłącz może odbywać
się przez zwieranie odpowiednich styków za pomocą przekaźnika. Bardziej
zaawansowane sterowniki umożliwiają nie tylko zdalną kontrolę parametrów
pracy, ale także diagnostykę urządzenia. Czasami do nawiązania komunikacji
wymagane jest podłączenie do sterownika dodatkowego modułu komunikacyjnego.

Często wykorzystywanym protokołem komunikacji z urządzeniami jest Modbus.
Modbus jest prostym i niezawodnym protokołem zapewniającym komunikację między
wieloma urządzeniami w architekturze master slave [7]. Komunikacja odbywa
się przeważnie przez interfejs szeregowy RS232 [8] lub RS485 [9]
(istnieje również odmiana protokołu nazwana Modbus TCP zapewniająca
komunikację przez sieć TCP/IP). W celu podłączenia Raspberry Pi do magistrali
RS232 lub RS485 niezbędny będzie odpowiedni moduł komunikacyjny (UART-RS232
lub UART-RS485). Obsługę łączności z urządzeniami zewnętrznymi można
zrealizować za pomocą biblioteki pymodbus [10], która oferuje pełną implementację
protokołu Modbus w języku Python.

## Podsumowanie
Technologia odgrywa coraz większą rolę w naszym życiu, a możliwości, które
oferuje nam w zakresie automatyzacji budynków, niewątpliwie staną się
powszechne w najbliższej przyszłości. Okazuje się, że już teraz możemy
uczestniczyć w rozwoju instalacji inteligentnych budynków wykorzystując dobrze
znane nam narzędzia.

## Bibliografia
1. Oficjalna strona MicroPython i PyBoard. http://www.micropython.org
2. Wiki ESP8266. https://nurdspace.nl/ESP8266
3. Repozytorium zawierające kod źródłowy portu MicroPython dla układu ESP8266. https://github.com/micropython/micropython/tree/master/esp8266
4. Narzędzie do komunikacji z bootloaderem układu ESP8266. \hyphenatedurl{https://github.com/themadinventor/esptool}
5. Pierwsze kroki z MicroPython i ESP8266. \hyphenatedurl{http://docs.micropython.org/en/latest/esp8266/esp8266/tutorial/index.html}
6. Oficjalna strona Raspberry Pi. https://www.raspberrypi.org
7. Podstawowe informacje o protokole Modbus. http://www.modbus.org/faq.php
8. Opis interfejsu RS232. https://pl.wikipedia.org/wiki/RS-232
9. Opis interfejsu RS485. https://pl.wikipedia.org/wiki/EIA-485
10. Implementacja protokołu Modbus dla Python. \hyphenatedurl{https://github.com/bashwork/pymodbus}
