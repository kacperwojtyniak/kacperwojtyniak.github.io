# Bezpieczne stringi

W poprzednich postach pokazałem jak można uzyskać dostęp do zasobów azurowych za pomocą Managed Identity. Jest to preferowany sposób uwierzytelniania i autoryzacji, jednakże nie zawsze jest on dostępny. Przykładowo nie dostaniemy się w ten sposób do Redisa. A może potrzebujemy skomunikować się z zewnętrznym API do którego mamy Basic token.

W takich przypadkach nie da się uniknąć sekretów. Gdzie je przechowywać? W Azure mamy do dyspozycji Azure Key Vault. Po szczegóły na temat tego czym jest ta usługa zapraszam do [dokumentacji](https://docs.microsoft.com/en-us/azure/key-vault/general/). Tutaj powiem tylko, że jest to rozwiązanie, które zapewni nam bezpieczne przechowywanie sekretów, certyfikatów czy kluczy. Pozwoli nam na ich rotację i do tego ma rozbudowany system uprawnień. A teraz zobaczmy w jaki sposób do owego Key Vaulta się podłączyć. W przykładach pokażę dostęp do danych typu `Secret` czyli wartości tekstowych jak connection stringi czy tokeny.

## Uprawnienia

Całe szczęście do samego Key Vaulta można się dostać używając Managed Identity. Tak więc zanim przejdziemy do przykładów wczytania sekretów, musimy nadać uprawnienia aplikacji która będzie z nich korzystać.

Po stworzeniu Key Vaulta należy przejść do zakładki `Access Policies` i wybrać `Add Access Policy`.

![dodanie_uprawnien](/images/bepieczne-stringi/1.png)

Następnie wybieramy jakie uprawnienia chcemy nadać oraz komu. Do mojego przykładu wystarczy mi czytanie sekretów. Zwróć uwagę jak duża jest tutaj granulacja. Możemy nadać uprawnienia tylko do odczytu albo tylko do zarządzania sekretami, albo tylko kluczami. Dzięki temu możemy zagwarantować, że każdy posiada jedynie minimalne wymagane uprawnienia.

![czytanie_sekretow_policy](/images/bepieczne-stringi/2.png)

Jako `service principal` wybrałem Service Principala o nazwie `accessakv` przypisanego do mojej aplikacji webowej. Oznacza to, że tylko kod działający w ramach tej konkretnej aplikacji będzie miał dostęp do sekretów.

![service_principal](/images/bepieczne-stringi/3.png)

## Dostęp bez zmian w kodzie

Zaskakującym może być to, że wcale nie trzeba wprowadzać modyfikacji w kodzie żeby uzyskać dostęp do sekretów. Wystarczy referencja w ustawieniach aplikacji na poziomie Azure portalu. Referencja taka ma następującą postać.

```
@Microsoft.KeyVault(SecretUri=https://<nazwa key vaulta>.vault.azure.net/secrets/<nazwa sekretu>/)
```

Do Key Vaulta dodałem sekret o nazwie `SecretString`.

![secret_string](/images/bepieczne-stringi/4.png)

W konfiguracji aplikacji dodałem więc referencję w postaci:

```
@Microsoft.KeyVault(SecretUri=https://mysecurestrings.vault.azure.net/secrets/SecretString/)
```

![konfiguracja](/images/bepieczne-stringi/5.png)

To już wystarczy, żeby w kodzie uzyskać dostęp do sekretnej wartości. W przypadku ASP.net od razu mam dostęp z pomocą interfejsu `IConfiguration`.

```csharp
public ConfigurationController(IConfiguration configuration)
{
    var secretValue = configuration["SecretString"];
}
```
W kodzie odwołujemy się już do nazwy z konfiguracji aplikacji a nie nazwy sekretu w w Key Vaulcie. W tym przykładzie w obu miejscach użyłem tej samej nazwy.

Więcej o tego typu referencjach poczytasz w [oficjalnej dokumentacji](https://docs.microsoft.com/en-us/azure/app-service/app-service-key-vault-references).

### ASP.Net i start aplikacji

Jeśli tworzysz aplikację w .net możesz w łatwy sposób wczytać sekrety podczas startu aplikacji. Wystarczy dodać kilka linijek w `Program.cs`. Potrzebna będzie jeszcze paczka nugetowa do [uwierzytelniania](https://www.nuget.org/packages/Azure.Identity) oraz dodania [configuration providera](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.Configuration.Secrets).

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            
                var builtConfig = config.Build();
                var secretClient = new SecretClient(
                    new Uri($"https://{builtConfig["KeyVaultName"]}.vault.azure.net/"),
                    new DefaultAzureCredential());
                config.AddAzureKeyVault(secretClient, new KeyVaultSecretManager());
            
        })
        .ConfigureWebHostDefaults(webBuilder => { webBuilder.UseStartup<Startup>(); });
```

Przykład jest zaczerpnięty z [dokumentacji](https://docs.microsoft.com/en-us/aspnet/core/security/key-vault-configuration?view=aspnetcore-5.0#use-managed-identities-for-azure-resources) i działa bezproblemowo.

W ten sposób mamy dostęp do sekretów tak jakbyśmy mieli je zapisane w pliku konfiguracyjnym `appsettings.json`, a więc poprzedni przykład z `IConfiguration` zadziała tak samo. 

### Mapowanie do klasy

Może być tak, że sekret jest częścią jakiejś sekcji konfiguracji. Przykładowo mamy klasę `MySettings` która wygląda tak:

```csharp
public class MySettings
{
    public string SecretSetting { get; set; }
    public string NotSoSecret { get; set; }
}
```

Właściwość `SecretSetting` jest zapisana w Key Vaulcie a `NotSoSecret` bezpośrednio w `appsettings.json`. W klasie `Startup.cs` dodajemy tą konfigurację jak gdyby nigdy nic.

```csharp
services.Configure<MySettings>(Configuration.GetSection("MySettings"));
```

Natomiast w Key Vaulcie sekret musimy nazwać `MySettings--SecretSetting`. 

## Na żądanie

Możemy oczywiście zażądać sekretu z kodu, nie w trakcie ładowania aplikacji. Nic prostszego.

```csharp
var client = new SecretClient(new Uri("https://securestrings.vault.azure.net/"), new DefaultAzureCredential());
KeyVaultSecret secret = client.GetSecret("SecretString");
string secretValue = secret.Value;
```

Bardziej rozbudowany tutorial znajdziesz [tutaj](https://docs.microsoft.com/en-us/azure/key-vault/general/tutorial-net-create-vault-azure-web-app).

## Podsumowanie

W żadnym z powyższych przykładów nie mamy do czynienia z surowym sekretem. Nigdy nie zostaje on zapisany w pliku na serwerze, nie musimy go kopiować, ustawiać w CI/CD itd. Wszystko jest po stronie Key Vaulta, bezpieczne, szyfrowane. Dostęp przy użyciu Managed Identity pozwala na uniknięcie jakiegokolwiek kopiowania sekretów oraz łatwą kontrolę uprawnień. Wszystko jest scentralizowane i zarządzane z jednego miejsca.

Jeśli jeszcze nie używasz Key Vaulta, connection stringi czy certyfikaty są kopiowane z miejsca w miejsce, powinieneś mocno się zastanowić nad zmianą. Jeśli obawiasz się o dostępność to niedawno [SLA wzrosło](https://azure.microsoft.com/en-gb/updates/akv-sla-raised-to-9999/) do poziomu 99.99%. Jestem przekonany, że masz większe szanse na wyciek swoich haseł niż na to, że Key Vault nie będzie działał.
