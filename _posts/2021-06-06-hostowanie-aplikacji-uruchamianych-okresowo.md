# Hostowanie aplikacji uruchamianych okresowo w Azure

W niemalże każdym projekcie można spotkać aplikacje albo skrypty które muszą być uruchamiane okresowo. Czasami jest to jakaś synchronizacja która wykonuje się co kilka godzin, czasami skrypt który musi coś posprzątać raz dziennie.

Spotkałem się już kilkukrotnie z pytaniem jak takie coś uruchomić w Azure. Opcji jest kilka i wybór będzie zależał od charakterystyki aplikacji którą chcemy uruchomić. Poniżej przedstawiam kilka możliwości.

## Azure Functions

Jest to opcja serverless. Oznacza to, że płacimy za czas działania naszej aplikacji lub skryptu. Możemy skorzystać z `Timer trigger` i za pomocą wyrażenia CRON wskazać kiedy funkcja ma się uruchamiać. W przypasku kodu C# będzie to wyglądało tak:

```csharp
[FunctionName("CronJob")]
public static void Run([TimerTrigger("0 */5 * * * *")]TimerInfo myTimer, ILogger log)
{
    
}
```
Funkcje można hostować na kilka sposobów. Consumption plan będzie dobry jeśli Twój kod wykonuje się krócej niż 10 minut. Jest to maksymalny czas trwania funkcji w planie consumption (domyślnie 5 minut). Koszty będą zależały od ilości wywołań i zużytych zasobów. Po szczegóły odsyłam do [dokumentacji](https://azure.microsoft.com/en-us/pricing/details/functions/)

W planie premium nie ma już ograniczenia czasowego, natomiast koszty są znaczące. Według kalkulatora Azure, minimalny koszt miesięczny to 155 dolarów. Jeśli miałbyś uruchamiać na tym tylko jeden kawałek kodu co kilka godzin, to może się okazać nieopłacalne.

Możliwe jest także hostowanie funkcji na app service planie. Jeśli nie potrzebujesz dużej mocy obliczeniowej, koszty względem planu Premium spadną znacząco. Przykładowo plan S1 to koszt rzędu 73 dolarów miesięcznie. To jest dobra opcja i łatwa w implementacji, pod warunkiem, że:
- wykorzystujesz app service plan do innych aplikacji, lub
- Twój "cron job" wykonuje się na tyle często, że nie marnujesz zasobów.

Zwracam uwagę na punkt drugi. Jeśli uruchamiasz joba np. raz dziennie, trwa to 30 minut, to przy tej opcji hostowania przez pozostałe 23h 30m płacisz za zasoby których nie używasz.

## Azure Web Jobs

Web joby to część App Service. Jeśli hostujesz już jakąś aplikację w App Service i masz tam wolne zasoby to możesz dorzucić jeszcze web joby. Nie potrzebujesz tworzyć nowych zasobów bo już wszystko masz, nie ponosisz dodatkowych kosztów. Szczegóły znajdziesz [tutaj](https://docs.microsoft.com/en-us/azure/app-service/webjobs-create). 

Rozwiązanie to sprawdza się dobrze. Osobiście sprawdziłem je w boju i nigdy nie było problemów. Na jednym App Service możemy mieć wiele jobów, mogą być uruchamiane cyklicznie gdzie częstotliwość definiujemy poprzez wyrażenie CRON, mogą działać ciągle lub być uruchamiane manualnie. 

W web jobach możemy uruchamiać skrypty bashowe, cmd czy bat, czego nie zrobimy w funkcjach. Oprócz tego jest też .exe, python czy java. Pełna lista oczywiście jest w [dokumentacji](https://docs.microsoft.com/en-us/azure/app-service/webjobs-create#acceptablefiles).

Wybierając web joby trzeba uważać na dwie rzeczy:
- Jeśli współdzielisz App Service z inną aplikacją, web joby będą korzystały z tych samych zasobów tj. procesora czy pamięci. Jeśli web job potrzebuje na tyle dużo mocy, że zabrankie przepustowości dla głównej aplikacji działającej na tym App Service, możesz mieć problem. W takiej sytuacji powinieneś się zastanowić nad dedykowanym App Service dla web jobów.
- Tak samo jak w funkcjach na app service planie, tutaj również płacisz 24h/7. Jeśli joby działają tylko 1h w ciągu doby, pozostałe 23h się marnują.

Warto też wspomnieć, że web joby dostępne są tylko dla App Service na Windowsie.

## Logic Apps

To zupełnie inna bajka. Model rozliczania serverless, także płacimy za wykonanie, podobnie jak Azure Functions. Różnica jest taka, że w Logic Apps mamy rozwiązanie no-code. Akcje definujemy w graficznym kreatorze. Oczywiście to co wyklikamy można zapisać w postaci pliku `json` i dodać do repozytorium. Niedawno znacząco poprawiły się możliwości lokalnego developmentu, można nawet debugować co nie jest takie oczywiste. Nowości zostały ogłoszone podczas ostatniego Microsoft Build i możesz o nich przeczytać [tutaj](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/azure-logic-apps-announcement-ga-of-single-tenant-standard-sku/ba-p/2382460). 

Po szczegóły logic appsów odsyłam do [dokumentacji](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview) ponieważ tutaj chcę się skupić na rozwiązanich w których hostujemy własny kod. Powiem tylko, że moim zdaniem jest to narzędzie przez wielu niedoceniane i serio warto się temu przyjrzeć. 

## Azure Container Instances

Jak sama nazwa wskazuje, jest to rozwiązanie w którym uruchamiamy kontener. Moim zdaniem jest to najciekawsze rozwiązanie, co nie znaczy że najlepsze. Jak wszystko, ma swoje wady i zalety. Jest też chyba najmniej znane.

Tak jak w funkcjach mamy tutaj model serverless. Płacimy tylko za czas działania kontenera. Znaczący plus względem App Service, ponieważ nie marnujemy zasobów kiedy kod nie działa. Jednocześnie nie mamy tutaj limitu czasu działania kontenera. Może on działać tak długo jak chcemy. Fakt konteneryzacji powoduje, że możemy uruchomić cokolwiek. 

Powiedziałem, że płacimy za czas działania. Ile ten czas kosztuje będzie zależało od tego jak dużo mocy przydzielimy. Tworząc Azure Container Instances Group wybieramy ile chcemy CPU oraz ile RAMu. Przykładowo jeśli nasz kontener będzie miał 1GB RAMu i jeden rdzeń, będzie uruchamiany raz dziennie na 60 minut, to w ciągu miesiąca będzie działał przez 108000 sekund co da nam 1.35 dolara miesięcznie.

[![aci_price](/images/hostowanie-aplikacji-uruchamianych-okresowo/1.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/1.png)

Oczywiście nie jest to rozwiązanie bez wad. Przy Container Instances komplikujemy nieco infrastrukturę. Potrzebujemy utworzyć Azure Container Instances, container registry na nasze obrazy dockerowe (na przykład Azure Container Registry), coś co będzie cyklicznie uruchamiać kontener (na przykład Azure Logic Apps). To ostatnie może nie być oczywiste a więc wyjaśnię. Jeśli kontener będzie np. aplikacją konsolową która wykona swoją robotę i się zamknie, to kontener umiera i przestajemy płacić, zasoby zostają zdealokowane. Potrzebne będzie jakieś zewnętrzne narzędzie żeby go cyklicznie uruchamiać. Dobrze sprawdzi się Logic App.

Warto zainteresować się tą usługą. Szczegóły oczywiście znajdą się w [dokumentacji](https://docs.microsoft.com/en-us/azure/container-instances/). Jeśli chcesz zobaczyć jak Container Instances działają w praktyce, to przygotowałem krótkie demo.

### Demo

Przygotowałem [repozytorium](https://github.com/kacperwojtyniak/aci-demo) które zawiera:
- przykładową aplikację Hello World działającą przez 5 minut
- Dockerfile wygenerowany z Visual Studio
- skrypt Powershell wykorzystujący moduł Az, który
  - utworzy resource grupę
  - utworzy container registry
  - zbuduje obraz dockerowy aplikacji demo
  - wypchnie obraz do container registry
  - utworzy Azure Container Instances Group wskazując uprzednio zbudowany obraz z tagiem `latest`

Po uruchomieniu skryptu `deploy_demo.ps1` będziesz miał działające ACI, z kontenerem który nie robi nic oprócz wypisania `Hello World` oraz `Finished` po upływie 5 minut. Kontener wystartuje od razu i po 5 minutach się zamknie. Od tego czasu nie ponosisz kosztów związanych z ACI. Opłaty za container registry będą się oczywiście naliczały.

W portalu możemy zobaczyć aktualny stan kontenera oraz np. logi.

[![container](/images/hostowanie-aplikacji-uruchamianych-okresowo/2.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/2.png)

### Cykliczne uruchamianie kontenera

Żeby cyklicznie uruchamiać kontener, możemy wykorzystać Logic Apps. Tego już nie zautomatyzowałem ale pokażę Ci jak możemy wszystko wyklikać w kilku krokach w portalu.

Najpierw tworzymy nowy zasób, wybierając Logic App, standardowo wybieramy nazwę oraz region. Kiedy zasób zostanie stworzony, przechodzimy do Logic Appa i tam możemy wybrać gotowy szablon `Recurrence`.

[![logic_app_template](/images/hostowanie-aplikacji-uruchamianych-okresowo/3.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/3.png)

Zobaczymy coś takiego.

[![logic_app_start](/images/hostowanie-aplikacji-uruchamianych-okresowo/4.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/4.png)

Tutaj należy określić jak często Logic App ma się uruchamiać. Nastepnie należy dodać kolejny krok. Wyszukujemy `Azure Container Instance` i wybieramy akcję `Start containers in a container group`. 

[![step](/images/hostowanie-aplikacji-uruchamianych-okresowo/5.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/5.png)

W kolejnym kroku wystarczy się zalogować, wybrać subskrypcję, resource grupę i wpisać nazwę ACI (skrypt domyślnie tworzy `aci-long-job`).

[![done_logic_app](/images/hostowanie-aplikacji-uruchamianych-okresowo/6.png)](/images/hostowanie-aplikacji-uruchamianych-okresowo/6.png)

Na koniec klikamy `Save` i od tego czasu Logic App będzie cyklicznie uruchamiał kontener.

## Podsumowanie

W tym wpisie nie pokazłem wszystkich opcji jakie daje nam Azure. Moglibyśmy przecież równie dobrze uruchamiać CRON joba w klastrze Kubernetesa albo na maszynie wirtualnej. Skupiłem się na kilku podstawowych usługach PaaS lub serverless. Każde rozwiązanie ma swoje plusy i będzie pasowało do innego scenariusza. Mam nadzieję, że dałem Ci ogólny pogląd na dostępne opcje, a dalej będziesz mógł drążyć temat samodzielnie.

