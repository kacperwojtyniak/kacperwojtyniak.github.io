# Azure Functions i .net 5

Microsoft oficjalnie udostępnił możliwość uruchamiania Azure Functions w .necie 5. Trochę to trwało, ale jest. Zaszły przy tym duże zmiany w sposobie hostowania, przez co nie wystarczy podnieść naszego projektu do nowej wersji .net. Trzeba podłubać troszkę więcej.

## Co się zmieniło

Do tej pory nasze funkcje były uruchamiane w tym samym procesie co host Azure Functions. Właśnie z tego względu wsparcie dla .net 5 nie było dostępne od razu. Nasz kod musiał być w tej samej wersji .neta co host, a więc trzeba było czekać aż Microsoft zacznie wspierać nowe wersje. 

Od teraz host oraz nasza aplikacja będą działały w osobnych, niezależnych procesach. Tak będzie dla .neta 5. W kolejnej wersji która ma się ukazać w listopadzie 2021 będziemy mieli możliwość działania "po staremu" lub już w odrębnym procesie. Od wersji .net 7 ma zniknąć możliwość współdzielenia procesu hosta.

## Ograniczenia

Jak wspomniałem, .net 6 będzie nam dawał wybór sposobu hostowania. To dlatego, że jeszcze nie wszystko działa w modelu "out of process". Główne braki, czy ograniczenia to:

- Brak wsparcia dla durable functions - ma być od .net 7
- Brak niektórych bindingów
- Ograniczenia w typach obiektów w bindingach - żegnamy się z DocumentClient czy CloudBlockBlob i pozostają nam stringi, tablice oraz zwykłe POCO.
- Dłuższy czas zimnego startu - przynajmniej teoretycznie, a jak duży jest to problem to się okaże.

## Zalety

Taki model hostowania ma niewątpliwie klika zalet.

- Uniezależnienie się od wersji hosta - dzięki temu mamy większą kontrolę nad naszą aplikacją i unikamy potencjalnych konfliktów w wersjach bibliotek.
- Szybsze wsparcie dla nowych wersji .neta
- Możliwość używania własnych middleware - moim zdaniem to może się okazać bardzo przydatne w niektórych przypadkach.

## Migracja istniejącego projektu

Przyjrzymy się testowemu projektowi, który posiada dwie funckje:

- Wywoływana cronem
- Wywoływana żądaniem http

Przyznam, że są to bindingi najczęściej przeze mnie używane i dlatego chcę się na nich skupić. Migracja pozostałych typów nie powinna przysparzać kłopotów. Należy jednak pamiętać, że nie wszystko jeszcze jest dostępne.

### Wersja .net

Pierwszym krokiem w procesie migracji będzie podniesienie projektu do .net 5. Początek naszego projektu .csproj będzie wyglądał tak:

```xml
<PropertyGroup>
  <IsPackable>false</IsPackable>
  <TargetFramework>net5.0</TargetFramework>
  <LangVersion>preview</LangVersion>
  <OutputType>Exe</OutputType>
  <AzureFunctionsVersion>v3</AzureFunctionsVersion>
</PropertyGroup>
```

### Paczki nuget

Czas na paczki nugetowe. Najpierw należy usunąć  `Microsoft.NET.Sdk.Functions`, a następnie dodać `Microsoft.Azure.Functions.Worker.Sdk` oraz `Microsoft.Azure.Functions.Worker`. Są to paczki potrzebne do działania w wyizolowanym procesie. 

To nie wszystko, bo tym razem nie dostajemy bindingów od razu jak to miało w poprzednich wersjach Azure Functions. Żeby nasza funkcja http mogła działać musimy dodać paczkę `Microsoft.Azure.Functions.Worker.Extensions.Http`, a TimerTrigger znajdziemy w `Microsoft.Azure.Functions.Worker.Extensions.Timer`.

### Kod funkcji

Sam kod funkcji też trochę się zmienia. Po pierwsze należy zmienić atrybut `FunctionName` na `Function`.
W funkcji z triggerem http zmienia się `HttpRequest` na `HttpRequestData`, oraz `IActionResult` na `HttpResponseData`. Do tego, w każdej funkcji `ILogger` musimy zastąpić kontekstem funkcji, tj. `FunctionContext` z którego możemy dostać loggera. Funkcja po staremu wygląda tak:

```csharp
[FunctionName("Hello")]
public IActionResult Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = "hello/{name}")] HttpRequest req,
    string name,
    ILogger log)
{
    log.LogInformation("Processed function in .net core");
    string responseMessage = $"Hello {name}. Welcome to .net core 3 functions.";
    return new OkObjectResult(responseMessage);
}
```
A po nowemu tak:

```csharp
[Function("Hello")]
public HttpResponseData Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = "hello/{name}")] HttpRequestData req,
    string name,
    FunctionContext ctx)
{
    ctx.GetLogger("hello").LogInformation("Processed function in .net core"));
    string responseMessage = $"Hello {name}. Welcome to .net 5 functions.";
    var response = req.CreateResponse(HttpStatusCode.OK);
    response.WriteString(responseMessage);
    return response;
}
```

W przypadku crona zmienia się tak samo atrybut `FunctionName`, brak loggera i nie mamy dostępu do klasy `TimerInfo`. Jeśli chcemy mieć dostęp do tych informacji które były do tej pory zawarte w  `TimerInfo` musimy sobie taką klasę stworzyć sami. W kolejnej wersji biblioteki `TimerInfo` ma już być dodane. Stary kod wygląda tak:

```csharp
[FunctionName("Cron")]
public void Cron([TimerTrigger("0 */1 * * * *")] TimerInfo myTimer, ILogger log)
{
    log.LogInformation("Hello from cron .net core 3");
}
```

A nowy, bez `TimerInfo` tak:

```csharp
[Function("Cron")]
public void Cron([TimerTrigger("0 */1 * * * *")] FunctionContext ctx)
{
    ctx.GetLogger("cron").LogInformation("Hello from cron .net 5");
}
```

### Program.cs

Musimy dorzucić jeszcze punkt startowy naszego programu w postaci pliku `Program.cs` z metodą `Main`.

```csharp
using Microsoft.Extensions.Hosting;

namespace Net5Functions
{
    class Program
    {
        public static void Main()
        {
            var host = new HostBuilder()
                .ConfigureFunctionsWorkerDefaults()
                .Build();

            host.Run();
        }
    }
}
```
W tym miejscu możemy też rejestrować zależności, tak jakbyśmy to zrobili w asp.net.

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(s =>
    {
        s.AddSingleton<IMyInterface, MyImplementation>();
    })
    .Build();
```

### Runtime

Na koniec zmieniamy runtime na isolated. W pliku `local.settings.json` ustawiamy:

```json
"FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
```

Należy pamiętać o tym ustawieniu hostując funckję w Azure. Też wymagane jest ustawienie `dotnet-isolated`.

## Debugowanie

Możliwe jest debugowanie funkcji lokalnie, ale póki co jest to nieco bardziej skomplikowane niż wciśnięcie F5. Musimy zainstalować [Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#v2), następnie z linii komend należy przejść do katalogu zawierającego nasz projekt i wywołać:

```
func start --dotnet-isolated-debug
```

Funkcja się zbuduje i uruchomi w debugu, a w konsoli powinniśmy zobaczyć:

```
[2021-03-16T15:28:17.248Z] Azure Functions .NET Worker (PID: 5432) initialized in debug mode. Waiting for debugger to attach...
```

w tym momencie w Visual Studio wybieramy z menu `Debug` opcję `Attach to process...` i szukamy naszego procesu po numerze PID.

## Linki

- [Jak stworzyć nowy projekt Azure Functions w NET 5](https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-developer-howtos?pivots=development-environment-vs&tabs=browser)
- [Dokumentacja na temat funkcji w wyizolowanym procesie](https://docs.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)
- [Paczki zawierające bindingi](https://www.nuget.org/packages?q=Microsoft.Azure.Functions.Worker.Extensions)
