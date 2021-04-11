# Dopasuj rozwiązanie techniczne do problemu a nie odwrotnie.

Tytuł tego posta to niby oczywista oczywistość. Jak porozmawiasz z jakimkolwiek developerem, to powie Ci, że tak właśnie robi, no bo jak to inaczej? Przecież zawsze zaczynamy od jakiegoś problemu i szukamy jego rozwiązania. Czy na pewno? Mam wrażenie, że szybko przechodzimy do momentu w którym mamy już wybrane rozwiązanie i próbujemy je uzasadnić w kontekście naszego problemu.

Jako zespół macie jakiś problem techniczny do rozwiązania, albo po prostu projektujecie coś od nowa. Zaczynacie myśleć. Pojawia się cała masa pomysłów, przytaczane są najlepsze praktyki, odkrywane są kolejne czynniki które trzeba wziąć pod uwagę, kolejne przypadki na które trzeba się zabezpieczyć. Zakres prac potrafi urosnąć bardzo szybko. Okazuje się, że kod to właściwie cały by trzeba przepisać, trzeba dołożyć co najmniej kilka nowych elementów infrastruktury ale to przecież nie problem bo to tylko kilka kliknięć w chmurze. A jak już przy tym jesteśmy to wypada się zeskalować do wielu regionów no bo niezawodność, bezpieczeństwo etc. Warto się zabezpieczyć na dużo większy ruch niż mamy obecnie. Najlepiej taki 100 razy większy.

Niestety bardzo łatwo wpaść w taką pułapkę i ja sam też w nią wpadam. Większość z nas lubi nowe zabawki i fajnie by było dołożyć do naszego systemu przynajmniej jakiś jeden ciekawy element. To może być cokolwiek i dla niektórych to może być serverless, dla innych kontenery, ktoś będzie chciał wrzucić uczenie maszynowe. To wszystko jest fajne, ciekawe, rozwojowe i na pewno trzeba uczyć się nowych rzeczy, tylko one nie zawsze są potrzebne. Tak samo jak nie używasz TIRa żeby przywieźć zakupy z marketu albo nie masz w domu gastronomicznego ekspresu do kawy, tak samo nie każdy technologiczny bajer potrzebny jest w Twoim systemie.

Nadmierne komplikowanie systemu niesie za sobą konsekwencje, o których często nie myślimy. Przede wszystkim ktoś to wszystko będzie musiał utrzymywać. Dokładanie kolejnych elementów infrastruktury spowoduje zwiększenie złożoności, a przez to trudniej będzie nowej osobie wdrożyć się w projekt. Ba, nawet Tobie z czasem może być trudno wszystko ogarnąć. Trzeba pamiętać, że sam kod też będzie bardziej złożony. Kolejna sprawa to testy. Wypada je mieć i utrzymywać. Komplikują się nie tylko testy jednostkowe ale też testy infrastruktury. Jest więcej rzeczy do ustawienia, więcej do przetestowania, może trzeba zrobić jakiś chaos engineering. To zaś przekłada się na czas potrzebny do wdrożenia rozwiązania, testowania i utrzymania, a jak wiemy czas przekłada się na pieniądze. Do kosztu czasu dokładamy koszty samej infrastruktury i może się okazać, że po prostu nie dostaniemy zgody na pracę nad takim rozwiązaniem.

Jak w takim razie nie zapędzić się i nie podążać tylko za kolejnymi fajnymi bajerami? Najważniejsze to najpierw zastanowić się co chcemy osiągnąć i dlaczego. Przykładowo mamy za zadanie zwiększyć niezawodoność naszego systemu. Moglibyśmy od razu wziąć się do roboty i np. uruchomić naszą aplikację w wielu regionach zamiast jednego. Zwiększa to niezawodność? Nie wiem. Najpierw trzeba odpowiedzieć na pytanie czym ta niezawodność jest. Powiedzmy, że znaczy to tyle, że użytkownik może otworzyć naszą aplikację, strona się wczytuje. No dobra, to w takim razie skąd będziemy wiedzieli, że niezawodność wzrosła? Ano trzeba najpierw zmierzyć jaką mamy teraz, jak często użytkownik nie może załadować strony. Może się okazać, że już jest na tyle wysoka, że jest wystarczająca. Nigdy nie będzie 100%. Podobnie można podejść do wydajności. Jeśli chcemy ją zwiększyć, to musimy wiedzieć w jaki sposób ją określić, jaką mamy teraz, jaki jest nasz cel.

Kiedy odpowiemy już na pytania co chcemy osiągnąć i mamy to mierzalne, wtedy możemy się zastanowić jak to zrobić. Moim zdaniem warto myśleć szeroko, wziąć pod uwagę różne, bardziej i mniej skomplikowane rozwiązania. Warto zrobić jakiś research i spisać jego wyniki. Nawet jeśli w tej chwili wybierzemy wersję minimum, to być może kiedyś będzie trzeba wrócić do tematu i wybrać coś innego. Wtedy przeprowadzony już research nie pójdzie na marne. Na końcu wybrać to, co po pierwsze umożliwi nam osiągnięcie celu. W trakcie obmyślania planu pewne rozwiązania mogą odpaść. Dopiero później wybieramy konkretne rozwiązanie i tutaj należy wziąć pod uwagę między innymi czynniki takie jak: 
- czas potrzebny do wdrożenia - jeśli coś będzie super wypaśne ale zajmie bardzo dużo czasu, to może nie jest to właściwa droga
- jak rozwiązanie wpłynie na złożoność systemu, jego testowalność, próg wejścia
- ryzyko - jeśli przebudujemy dużą część systemu, wdrożenie może być ryzykowne
- koszty - zarówno czasu pracy, infrastruktury, późniejszego utrzymania
- wiedzę i doświadczenie członków zespołu - jeśli w zespole tylko jedna osoba zna daną technologię, może powinniśmy wybrać coś innego
Biorąc te wszystkie czynniki pod uwagę, może się okazać, że nie potrzebujemy rozwiązań na miarę Netflixa a i tak osiągniemy nasze cele.

Jak już wspomniałem, każdy, albo prawie każdy z nas lubi nowe zabawki. Trudno się pochamować przed sięgnięciem po "latest and greatest", ale uważam, że każdy inżynier musi się tego nauczyć. Zawsze należy zastanowić się co chcemy osiągnąć, a nie wybierać zabawki i próbować uzasadnić ich wybór.