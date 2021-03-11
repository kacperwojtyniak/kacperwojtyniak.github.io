# Tniemy koszty w Azure

![money](/images/money.jpg)

W chmurze na wyciągnięcie ręki mamy ogromne moce obliczeniowe, setki usług ze wsparciem dostawcy, wysoką dostępność i najnowsze zabawki. Kilkoma kliknięciami tworzymy konto, zakładamy subskrypcję, tworzymy zasoby i płacimy tylko za to, czego używamy. To wszystko takie piękne, nowoczesne, szybkie. Łatwo jest tworzyć i niestety równie łatwo można skończyć z ogromnymi rachunkami. Wystarczy, że zapomnimy o skasowaniu jakiejś maszyny wirtualnej, omyłkowo wybierzemy drogą wersję usługi albo nie zrozumiemy za co tak naprawdę płacimy. O tym chyba już każdy wie i stara się trzymać koszty w ryzach. Nie każdy jednak wie, że oprócz odpowiedniego dobierania zasobów do potrzeb, powinniśmy również dobrać sposób rozliczania się. 

W Azure mamy dwie opcje dla małych lub średnich firm na zredukowanie cen.

## Subskrypcja Dev/Test

Domyślną subskrypcją w Azure jest Pay-As-You-Go. Jeśli masz prywatne konto Azure to jest duża szansa, że korzystasz właśnie z niej. Wystarczy jednak, że wykupiłeś subskrypcję na Visual Studio Professional lub Enterprise i możesz założyć drugą subskrypcję, tym razem typu Pay-As-You-Go Dev/Test. Kiedy mówimy o firmie w której wykorzystywany jest .net, istnieje duża szansa, że używacie Visual Studio i taka subskrypcja jest dostępna.

Jak sama nazwa wskazuje, subskrypcja ta idealnie nadaje się do developmentu i testowania. Dlaczego? Bo jest dużo taniej. Niestety nie wszystko jest tańsze, więc oszczędności będą w dużej mierze zależały od tego jakich zasobów potrzebujesz. Przykładowo, maszyny wirtualne z Windowsem kosztują tyle samo co te z Linuxem. Oznacza to, że goła maszyna wirtualna D2v3 zamiast kosztować $152.62 miesięcznie, kosztuje $85.46. App service S1 zamiast kosztować $73 miesięcznie, kosztuje $43.80. Microsoft po prostu nie pobiera opłaty za Windowsa.

Jak widać w powyższych przykładach oszczędności są spore. Należy jednak pamiętać, że oszczędności różnią się w zależności od usługi, wybranego rozmiaru itd. Niższe ceny oferowane są dla:
 - Windows Virtual Machines
 - BizTalk Virtual Machines (Enterprise oraz Standard)
 - Azure SQL Database
 - SQL Server Virtual Machines (Enterprise, Standard oraz Web)
 - Logic Apps Enterprise Connection
 - App Service (Basic, Standard, Premium v2, Premium v3)
 - Cloud Services instances
 - HDInsight instances

Zgodnie z zasadami użycia, takiej subskrypcji nie możemy wykorzystywać do tworzenia środowisk produkcyjnych, nie ma gwarantowanego SLA a Microsoft może zawiesić działanie dowolnego zasobu jeśli jest uruchomiony dłużej niż 120 godzin lub uzna, że jest to środowisko używanie produkcyjnie. Nie znaczy to, że każdy zasób co działa ponad 120 godzin będzie zawieszony. Jest to raczej furtka dla Microsoftu.

Żeby sprawdzić ceny dla subskrypcji Dev/Test, w [kalkulatorze Azure](https://azure.microsoft.com/en-us/pricing/calculator/) należy kliknąć odpowiednią opcję.

![calculator](/images/dev-test-pricing.png)

Więcej informacji na temat Dev/Test znajdziesz [tutaj](https://azure.microsoft.com/en-gb/offers/ms-azr-0023p/)

### Dodatkowe korzyści osobnych subskrypcji

Oprócz oczywistych korzyści finansowych, osobna subskrypcja wykorzystywana do środowisk nieprodukcyjnych oraz testów, umożliwi lepszą kontrolę uprawnień oraz kosztów. 

W subskrypcji podstawowej gdzie stoi produkcja, można ograniczyć uprawnienia do tworzenia, modyfikowania, usuwania zasobów tak, aby mieć pewność że nikt produkcji nie zepsuje. Natomiast w Dev/Test możemy ograniczenia rozluźnić i pozwolić developerom, testerom na tworzenie zasobów wedle potrzeb.

Mając taki rozdział środowisk na subskrypcje, łatwo będziemy mogli zobaczyć ile kosztuje nas produkcja a ile development. To może ułatwić poszukiwanie kolejnych oszczędności.


## Płatność z góry

Płacenie za zasoby z miesiąca na miesiąc jest super. Używamy czegoś - płacimy. Nie używamy, usuwamy - nie płacimy. Co jeśli mamy pewne zasoby które mamy zawsze i to się na pewno nie zmieni? Możemy wtedy zapłacić z góry za dłuższy okres i otrzymać pokaźny rabat.

Przykładowo mamy aplikację, która działa już jakiś czas, biznes się rozwija a my wiemy, że jeszcze kolejny rok App Service na którym jest hostowana będzie nam potrzebny. Możemy wtedy wykupić App Service na rok z góry. Zobowiązujemy się wtedy do płatności miesięcznych przez rok lub płatności jednorazowej za cały okres. Druga opcja to wykupienie aż trzech lat. Wtedy rabat jest jeszcze wyższy. W przypadku App Service P1V3 rabaty to 25% przy rocznej rezerwacji lub 40% jeśli zdecydujemy się na trzy lata. Dla innych zasobów rabaty są różne. Nie sprawdzałem wszystkich ale rzucając okiem na kilka z nich, przy rocznym zobowiązaniu możemy liczyć na około 30% a przy trzyletnim na około 50% upustu.

Pełną listę zasobów z możliwością rezerwacji i warunki znajdziesz [tutaj](https://docs.microsoft.com/en-us/azure/cost-management-billing/reservations/prepare-buy-reservation?toc=/azure/cost-management-billing/reservations/toc.json)

## Podsumowanie

Jak widać w łatwy sposób możemy obniżyć koszty usług w Azure. Wielkość rabatu będzie zależała od rodzaju usług i ich rozmiarów. Mimo wszystko warto to rozważyć jeśli mamy stałe środowiska nie będące produkcją. Dzięki temu zyskamy także możliwość lepszego zarządzania uprawnieniami a to pozytywnie wpłynie na aspekt bezpieczeństwa.
Jeszcze łatwiej obniżyć koszty wybierając płatność z góry za dłuższy okres, natomiast tutaj musimy być pewni, że to się opłaci.

Zachęcam do sprawdzenia jak to wygląda u Ciebie w firmie. Może przyczynisz się do obniżenia kosztów?