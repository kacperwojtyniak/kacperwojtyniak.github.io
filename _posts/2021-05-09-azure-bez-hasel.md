# Azure bez haseł

Podejrzewam, że nikt z nas nie lubi zarządzania hasłami, kluczami, tokenami czy connection stringami. Całe szczęście w niektórych przypadkach możemy tego uniknąć. Nie znaczy to, że mamy zdjąć wszystkie zabezpieczenia. Co to, to nie. Wręcz przeciwnie. Pozbywając się haseł możemy zwiększyć bezpieczeństwo naszych systemów. Posłuży nam do tego Azure Active Directory oraz Managed Identities.

Pokażę Ci Managed Identity na przykładzie aplikacji w .net core, z poziomu której uzyskam dostęp do dwóch zasobów:
- Azure Service Bus
- Azure Function zabezpieczonej poprzez Azure AD, co opisałem w [poprzednim wpisie](https://kacperwojtyniak.github.io/2021/05/02/uwierzytelnianie-no-code.html)
  
Wybrałem te dwa przykłady, ponieważ pokrywają wszystkie możliwe scenariusze, tj. dostęp do usługi zarządzanej przez Azure oraz do naszego własnego API. Pełną listę usług które wspierają uwierzytelnianie z użyciem Managed Identity znajdziesz [tutaj](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-managed-identities#azure-services-that-support-azure-ad-authentication). Lista nie jest długa, ale jest tam kilka kluczowych zasobów.

## Czym jest Managed Identity

Identity jak pewnie wiesz, to tożsamość. Managed to zarządzana. A więc jest to tożsamość którą w tym przypadku zarządza Azure, a my nie musimy się o nic martwić. Włączając Managed Identity dla naszej maszyny wirtualnej albo aplikacji webowej, w naszym Azure Active Directory tworzony jest "service principal". To jemu możemy nadawać uprawnienia np. prawo do wysyłania wiadomości do Service Busa albo ich odbierania. Service principal jest przypisany do danego zasobu np. wspomnianej maszyny wirtualnej. 

W skrócie znaczy to tyle, że nasz kod uruchomiony na zasobie mającym włączone Managed Identity może się odwołać do lokalnego adresu i uzyskać access token do jakiegoś zasobu np. Azure Key Vault lub Service Bus. My nie musimy w konfiguracji naszej aplikacji ustawiać żadnych haseł, tokenów czy innych sekretnych wartości. Oczywiście proces uzyskania takiego tokena jest przykryty biblioteką i sprowadza się do wywołania jednej metody.

To oczywiście bardzo powierzchowny opis. Jeśli chcesz się dowiedzieć więcej o tym jak cały mechanizm działa to zajrzyj do [dokumentacji](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/). W tym wpisie pokażę Ci jak z tego mechanizmu skorzystać.

## Włączenie Managed Identity 

Włączenie Managed Identity to nic prostszego niż przełączenie przysłowiowej wajchy w portalu. Oczywiście w prawdziwym świecie zalecane jest skorzystanie z jakiejś formy automatyzacji.

![włączenie_msi](/images/azure-bez-hasel/1.png)

Po kliknięciu "Save" dostaniemy informację, że nasza aplikacja zostanie zarejestrowana w Azure AD.

![azure_AD_rejestracja](/images/azure-bez-hasel/2.png)

To wszystko jeśli chodzi o "System Assigned Managed Identity". W przypadku "User Assigned" jest kilka kroków więcej. Od teraz możemy już prosić Azure AD o tokeny.

## Uwierzytelnianie w usługach Azure

Zobaczmy na przykładzie Azure Service Bus jak możemy się uwierzytelnić. Dla innych usług będzie to wyglądać tak samo.

Azure AD wyda nam już token, ale nie będzie on zawierał jeszcze żadnych ról. Jesteśmy uwierzytelnieni ale nie mamy autoryzacji do komunikacji z Azure Service Bus czy też innymi usługami. Żeby nadać uprawnienia idę do mojego Azure Service Bus w portalu, wybieram zakładkę `Access Control (IAM)` i dodaję rolę. W moim przypadku jest to `Azure Service Bus Data Sender` przypisana do App Service o nazwie `mycaller`. Umożliwi mi to wysyłanie wiadomości do Service Busa.

![rola_sender](/images/azure-bez-hasel/3.png)

Tyle wystarczy żeby kod działający w ramach App Service `mycaller` mógł wysyłać wiadomości do mojego Service Busa.

### Kod

W kodzie wygląda to bardzo prosto.

```csharp
var asb = new ServiceBusClient("msiexample.servicebus.windows.net", new DefaultAzureCredential());
var sender = asb.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("test"));
```

Zwróć uwagę na to, że nie ma tutaj connection stringa ani żadnego hasła. Podaję jedynie nazwę mojego Service Bus Namespace oraz tworzę obiekt klasy `DefaultAzureCredential`. Klasa ta pochodzi z nugeta `Azure.Identity`. Możesz stworzyć swoją klasę, która będzie dziedziczyć po `TokenCredential` i użyć jej w tym miejscu.

## Uwierzytelnianie i autoryzacja w naszych API

### Uwierzytelnianie
Równie łatwe jest uzyskanie access tokena do komunikacji z naszym własnym API.

```csharp
var applicationId = "821f5aba-e810-48bc-a4d1-75b0f9ca0093";
var tokenCredential = new DefaultAzureCredential();
var accessToken = await tokenCredential.GetTokenAsync(
        new TokenRequestContext(scopes: new string[] { $"{applicationId}/.default" })
    );
```

Jest to zwykły OAuth. Chcąc wysłać żądanie HTTP do zabezpieczonego API, token należy umieścić w nagłówku `Authorization: Bearer <token>`.

W powyższym przykładzie wartość `appliactionId` to `Application (client) ID` z `App Registration` w Azure AD, dla którego chcemy otrzymać token.

![app_id](/images/azure-bez-hasel/4.png)

### Autoryzacja

Mając uwierzytelnianie z głowy, możemy przejść do autoryzacji. Access token który otrzymałem w powyższym przykładzie nie zawiera żadnych ról. Możemy to zmienić w kilku krokach. 

Najpierw należy dodać rolę do App Registration. W tym przypadku w Azure AD znajduję aplikację do której będę wysyłał żądanie HTTP i przechodzę do zakładki `App Roles`, następnie dodaję rolę.

![rola](/images/azure-bez-hasel/5.png)

Kolejny krok to nadanie tej roli mojemu identity, tak żeby znalazła się w tokenie. Potrzebuję do tego kilku rzeczy:

- RoleId, czyli ID roli którą właśnie stworzyłem, dostępne z portalu zaraz po jej dodaniu.

![role_id](/images/azure-bez-hasel/6.png)
  
- ObjectId aplikacji enterprise
Tutaj uwaga żeby się nie pomylić. W Azure AD, znajdujemy app registration i przechodzimy do odpowiadającego Enterprise App Registration, klikając w miejscu zaznaczonym na poniższym obrazku.

![enterprise_app](/images/azure-bez-hasel/7.png)

Dopiero tutaj kopiujemy `ObjectId`.

![objectId](/images/azure-bez-hasel/8.png)

- ObjectId naszego Managed Identity

Najłatwiej je znaleźć przechodząc do miejsca w którym włączyliśmy Managed Identity.

![msi_objectId](/images/azure-bez-hasel/9.png)

- Zainstalowany moduł Powershella o nazwie `AzureAd`

Kiedy mamy wszystkie dane możemy nadać rolę przy użyciu powershella.

```powershell
Connect-AzureAd -TenantId <your Azure AD tenant ID>

New-AzureADServiceAppRoleAssignment -Id 2e7fa5b5-7d2c-4a4b-9b9e-685007caa557 `
     -PrincipalId 4c11fab0-73e2-45cc-863a-2368fb5c2aaa `
     -ObjectId 4c11fab0-73e2-45cc-863a-2368fb5c2aaa `
     -ResourceId 91982bcc-0503-47a2-b60b-2f533624f612
```

To wszystko. Tokeny powinny już zawierać rolę o nazwie `demo_role`.

Na pewno można się w tym wszystkim trochę pogubić. W szczególności która aplikacja w Azure AD jest od czego i które ID należy skopiować. Najlepiej jak przetestujesz sobie wszystko sam.

### Pułapka

Jeśli chcesz przejść przez te wszystkie kroki i tak jak ja otrzymać token do API zabezpieczonego w sposób opisany w [poprzednim wpisie](https://kacperwojtyniak.github.io/2021/05/02/uwierzytelnianie-no-code.html) czycha na Ciebie pewna zasadzka. Token otrzymasz, natomiast nie będzie uznany za prawidłowy. Zdekoduj token i zwróć uwagę na claim `iss`. Domyślnie będzie on wyglądał tak: `https://login.microsoftonline.com/<your tenant ID>/`. Jest to token w wersji 1.0. Kiedy włączyłeś uwierzytelnianie dla swojej aplikacji przez Azure Portal, automatycznie ustawiło się wymaganie tokena w wersji 2.0.

Możesz edytować wymaganego "issuera".
![edit_auth](/images/azure-bez-hasel/10.png)

Tutaj możesz usunąć "v2.0"
![edit_auth_2](/images/azure-bez-hasel/11.png)

W tym momencie już wszystko powinno grać.

Druga opcja to włączenie tokenów 2.0 w App Registration. W tym celu w Azure AD znajdź App Registration dla którego chcesz otrzymać token, edytuj manifest i ustaw `accessTokenAcceptedVersion: 2`. Wtedy wszystkie tokeny które otrzymasz dla tej aplikacji będą w wersji 2.0, tj. claim `iss` będzie miał końcówkę `/v2.0`.

## Podsumowanie
Ustawienie wszystkiego może i nie jest najprzyjemniejsze. Mimo wszystko jest tego warte. Na żadnym etapie konfiguracji nie mamy do czynienia z sekretnymi wartościami, nie musimy żadnych sekretów przechowywać, nie ma problemu z rotacją kluczy. Oprócz tego możemy w łatwy sposób zarządzać uprawnieniami na poziomie Azure AD. Uprawnienia można nadawać nie tylko aplikacjom, ale także użytkownikom lub grupom z Azure AD.


