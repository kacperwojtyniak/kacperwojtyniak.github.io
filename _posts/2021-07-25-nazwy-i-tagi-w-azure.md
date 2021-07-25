# Nazwy i tagi w Azure

Kilkukrotnie zauważyłem, że kiedy to programista zajmuje się środowiskami chmurowymi, organizacja zasobów pozostawia nieco do życzenia. O ile zazwyczaj jako programiści dbamy o odpowiednie nazwy metod, klas, itd. tak niekoniecznie przykładamy taką samą uwagę do infrastruktury. Kiedy zasobów jest jeszcze mało nie stanowi to problemu, natomiast kiedy organizacja rośnie, zasobów jest coraz więcej, pojawia się problem. Nagle nie wiadomo co jest czym, gdzie jest produkcja ani kto odpowiada za dany serwer. W tym wpisie chciałbym przedstawić kilka wskazówek dotyczących nazewnictwa i oznaczania zasobów w Azure.

## Jednolite i jednoznaczne nazewnictwo

Kiedy piszemy kod, zastanawiamy się jak powinna się nazywać dana klasa. Dzięki dobrej nazwie łatwo tą klasę odnajdziemy, oraz nie będziemy mieli wątpliwości do czego służy. Jeszcze ważniejsze jest to, że ktoś kto przyjdzie po nas również będzie wiedział co to jest. Takie same zasady powinny obowiązywać w przypadku nazywania zasobów w chmurze. 

Wyobraź sobie, że budujesz nową aplikację i jednym z wymogów jest składowanie jakichś raportów w formie plików zapisywanych w Azure Storage Account. W tym celu tworzysz nowy Storage Account o nazwie `reports`. Nazwa jest bardzo dobra. Jasno mówi co się tutaj znajduje.

Za jakiś czas przychodzi Ci napisać drugą aplikację, która również ma składować raporty w Storage Account, przy czym nie może to być ten sam zasób. Musi być w osobnym regionie i te raporty dotyczą czegoś zupełnie innego. Jak w takim przypadku nazwać nowy zasób? Może `reports2`? Pewnie nie. Prawdopodobnie skorzystasz z nazwy aplikacji i wyjdzie coś w stylu `salesreports`.

W tej sytuacji będzie jeszcze dosyć łatwo się połapać, ale takich zasobów z czasem może być jeszcze więcej i wtedy będzie kłopot.

### Strategia nazewnictwa

Żeby zapanować nad nazwami zasobów, powinieneś trzymać się jednej konwencji, której powinni używać wszyscy w firmie. Zastanów się, co pomoże Ci jasno identyfikować z czym dany zasób jest związany. Dotyczy to zarówno ResourceGroups jak i konkretnych usług. Dobrym punktem startowym będą wskazówki od Microsoftu które znajdziesz [tutaj](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)

Moim zdaniem nazwa powinna zawierać przynajmniej trzy elementy:
- nazwa aplikacji/projektu - łatwo powiążesz dany zasób z konkretnym projektem, będziesz wiedział do czego to służy
- skrót nazwy regionu, np. weu dla WestEurope - może się zdarzyć, że ta sama usługa będzie uruchomiona w kilku regionach i dzięki temu łatwiej się zorientujesz co jest czym
- nazwa środowiska - zazwyczaj środowisk jest kilka, jest produkcja, qa, dev, etc. Zawierając nazwę środowiska w nazwie zasobu oszczędzisz sobie stresu i łatwo zidentyfikujesz czy przypadkiem nie dłubiesz przy produkcji. (Warto nie produkcyjne zasoby wydzielić do osobnej subskrypcji, o czym wspominałem [w osobnym wpisie](https://kacperwojtyniak.github.io/2021/03/11/tniemy-koszty-azure.html))

Konwencję musisz oczywiście dostosować do własnych potrzeb. Najważniejsze to ustalić jeden szablon, który będzie używany przez wszystkich. 

## Tagi

Niektórzy twierdzą, że tagi są niepotrzebne. Moim zdaniem nie jest to prawda. Tagi mogą być świetnym narzędziem które pomoże w sprawnym zarządzaniu zasobami.

Odpowiednie nazewnictwo to podstawa, natomiast w nazwie nie zawrzemy wszystkich informacji. W tagach możemy umieścić cokolwiek uznamy za potrzebne. Dla mnie najistotniejsze są:
- nazwa środowiska - tak, tak, w nazwie zasobu też powinna być, natomiast umieszczenie jej w tagach ułatwi filtrowanie i raportowanie kosztów
- nazwa zespołu lub nazwisko osoby odpowiedzialnej za dany zasób - może się zdarzyć, że chcesz znaleźć właściciela zasobu i nie wiadomo gdzie go szukać, taki tag na pewno w tym pomoże
- nazwa projektu - w przypadku kiedy pod jeden projekt przypada wiele aplikacji, w dodatku rozsianych w różnych grupach czy regionach

Inne przykłdowe tagi które mogą Cię zainspirować do opracowania własnych wymagań możesz znaleźć [na stronach Microsoftu](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging).

Tagi pomogą nie tylko odnaleźć osobę odpowiedzialną, ale pomogą też w zarządzaniu. Dla mnie pomocne są przede wszystkim w dwóch sytuacjach.

### Analiza kosztów

Kiedy zajrzysz do swojej subskrypcji, w zakładce `Cost analysis` domyślnie zobaczysz koszty z podziałem na rodzaj zasobu, lokalizację i resource grupę. Możesz ten widok zmienić i zobaczyć podział na tagi. Jeśli każdy zasób będzie miał tag odpowiadający nazwie środowiska, będziesz mógł zobaczyć ile kosztuje produkcja a ile qa. Dzięki tagom łatwo sprawdzisz ile kosztują zasoby per projekt czy zespół. Wszystko jak na dłoni

[![koszty](/images/nazwy-i-tagi-w-azure/cost.png)](/images/nazwy-i-tagi-w-azure/cost.png)

### Filtrowanie

Zasobów w subskrypcji czy nawet w resource grupie może być całkiem sporo. Całe szczęście możemy zasobów szukać nie tylko po nazwie ale i po tagach.

[![filter](/images/nazwy-i-tagi-w-azure/filter.png)](/images/nazwy-i-tagi-w-azure/filter.png)

Filtrowanie pomocne było dla mnie szczególnie kiedy pewnego razu natrafiłem na resource grupę w której panował bałagan. W jednym miejscu mieszały się zasoby z różnych środowisk i trzeba było je przenieść do osobnych grup, a nawet subskrypcji. Zacząłem od analizy co jest czym, a następnie przypisałem zasobom odpowiednie tagi. Dzięki temu wiedziałem co należy przenieść i gdzie.

## Polityka

Łatwo powiedzieć, że chcemy mieć takie i takie nazwy, a każdy zasób ma mieć konkretne tagi. Trudniej wykonać. Jest to szczególnie problematyczne w organizacjach gdzie każdy członek zespołu jest w stanie tworzyć zasoby a zespoły same utrzymują i zarządzają środowiskami. Nie trudno o rozjazd pomiędzy zespołami i projektami.

Z pomocą przychodzą [polityki azura](https://docs.microsoft.com/en-us/azure/governance/policy/overview).

Polityki służą do utrzymania standardów. Nie tylko nazewnictwa. Dzięki politykom można wymusić używanie "private endpoint" dla Service Busa albo możemy przypilnować żeby klucze Storage Account nie straciły ważności. Możliwości jest mnóstwo. Wbudowanych polityk jest przeszło 700, a do tego możemy tworzyć własne. 

Dzięki politykom możemy ustalić, że każdy zasób musi posiadać konkretny tag, np. `environment`. Bez niego nie będzie możliwe stworzenie zasobu. Domyślnie istniejące już zasoby nie zostaną zmodyfikowane. Zostaną natomiast zgłoszone, że nie stosują się do polityki. Dzięki temu możemy je łatwo wychwycić i naprawić.

Na poniższym zrzucie ekranu widzisz, że tylko 2 zasoby spełniają politykę wymagającą wybraną politykę. Pozostałe zasoby mogę łatwo odszukać i naprawić.

[![compliance_state](/images/nazwy-i-tagi-w-azure/compliance_state.png)](/images/nazwy-i-tagi-w-azure/compliance_state.png)

Nie musimy iść tak bardzo po bandzie i zabraniać tworzenia zasobów niespełniających danej polityki. Możemy umożliwić ich tworzenie, natomiast będą zgłoszone jako te niestosujące danej polityki.

Podobnie możemy zrobić w temacie nazw. Tutaj sprawa jest nieco trudniejsza bo nie ma wsparcia dla regexa oraz musimy politykę zdefiniować sami. Jest to nieco uciążliwe, dlatego osobiście polegałbym na polityce tagów, a nazw pilnował ręcznie. W [dokumentacji](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure) znajdziesz informacje na temat własnych polityk.

## Podsumowanie

Nieważne jakie konkretnie przyjmiesz zasady nazewnictwa i tagowania. Ważne, żeby było to spójnie na przestrzeni całej organizacji i aby zarówno nazwy jak i tagi dostarczały niezbędnych informacji potrzebnych do identyfikacji zasobów. Ułatwi to zarządzanie, a także raportowanie i analizę kosztów. Przypilnowanie odpowiedniej konwencji nazw jest nieco trudniejsze, ale z tagi możemy sobie poradzić poprzez polityki, do czego zachęcam. 