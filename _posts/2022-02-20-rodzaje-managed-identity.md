# Rodzaje managed identity

W postach [Azure bez haseł](https://kacperwojtyniak.github.io/2021/05/09/azure-bez-hasel.html) oraz [Bezpieczne stringi](https://kacperwojtyniak.github.io/2021/05/19/bezpieczne-stringi.html) mówiłem o tym jak wyeliminować lub ograniczyć korzystanie z sekretów w Azurze. W tym celu posługiwałem się [Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview).
W tym wpisie chciałbym pokrótce powiedzieć Ci jakie wyróżniamy rodzaje Managed Identity oraz jak zdecydować który rodzaj wybrać.

## System assigned

Pierwszy, chyba najczęściej używany w przykładach, rodzaj Managed Identity to System Assigned. Jest to rodzaj identity które jest silnie powiązane z jednym zasobem np. z maszyną wirtualną albo app servicem, oraz jest tworzone (i usuwane) automatycznie przez Azura. Dla przykładu załóżmy że chcemy włączyć Managed Identity dla maszyny wirtualnej.

Możemy to zrobić na wiele sposobów - CLI, Powershell, Portal, itd. W Azure Portalu wygląda to tak:

[![enable-identity](/images/rodzaje-managed-identity/enable-identity.jpg)](/images/rodzaje-managed-identity/enable-identity.jpg)

Kiedy zapiszemy takie ustawienia, w Azure Active Directory powstaje Service Principal, który jest powiązany z tą jedną maszyną wirtualną. Kluczowe jest to, że z jedną. System Assigned identity nie może być przypisane do wielu różnych zasobów. Oprócz tego, dany zasób może posiadać tylko jedno System Assigned Managed Identity. W przypadku User Assigned już tak nie jest. 

Kolejna sprawa to cykl życia. System Assigned identity możemy włączyć dla istniejącego już zasobu a w momencie kiedy ten zasób usuniemy, razem z nim znika Identity. W niektórych przypadkach może to mieć znaczące konsekwencje. Przykładowo, maszyna wirtualna `A` posiada System Assigned Managed Identity. Temu Identity nadaliśmy rolę `Storage Blob Data Contributor` do jakiegoś storage account zarządzanego przez inny zespół. Z jakiegoś powodu usuwamy maszynę wirtualną i zastępujemy ją maszyną `B`. W tym momencie musimy od nowa nadać uprawnienia do storage account dla Identity maszyny `B`, ponieważ wraz z usunięciem maszyny `A` Identity również zostało usunięte. 

Podsumowując, System Assigned Managed Identity:
- Nie może być przypisane do wielu zasobów.
- Nie może istnieć bez zasobu. W związku z tym nie możemy mu przypisać roli zanim nie powstanie zasób.
- Jest usuwane wraz z usunięciem zasobu (tj. maszyny wirtualnej, app service, etc.)
- Jest tylko jedno per zasób. Przykładowo maszyna wirtualna nie może mieć więcej niż jedno System Assigned Managed Identity.


## User assigned managed identity

Funkcjonalnie User Assigned Managed Identity jest identyczne, tzn. wykorzystamy je do tego samego. Są jednak pewne różnice między User Assined a System Assigned w sposobie zarządzania nimi.

User Assigned Managed Identity jest pełnoprawnym zasobem w Azurze. Tworzymy je tak samo jak np. storage account i widzimy w naszej resource grupie.

[![user-assigned](/images/rodzaje-managed-identity/user-assigned.jpg)](/images/rodzaje-managed-identity/user-assigned.jpg)

Po stworzeniu takiego identity możemy je przypisać do wielu różnych zasobów. Możemy mieć App Service który ma kilka deployment slotów i każdemu z nich przypisać to samo User Assigned Managed identity. Dzięki temu uprawnienia, role, nadane dla tego Identity od razu są dostępne dla wszystkich deployment slotów. W przypadku System Assigned każdy Slot miałby swoje unikatowe Identity i każdemu nich należałoby nadać uprawnienia osobno. Podobnie możemy postąpić np. z grupą maszyn wirtualnych które powinny mieć te same uprawnienia - każdej maszynie w grupie przypiszemy to samo Identity.

Co więcej, nie tylko możemy jedno Identity przypisać wielu zasobom, jeden zasób może mieć wiele identity. Taki układ może niekiedy być wygodny z powodów organizacyjnych, zarządzania uprawnieniami. W takim przypadku aplikacja uruchomiona w zasobie z wieloma User Assigned Managed Identity, w momencie kiedy chce uzyskać token z Azure AD, musi wskazać którego identity chce użyć.

W dotnecie może to wyglądać tak.

```csharp
var credentialOptions = new DefaultAzureCredentialOptions()
{
    ManagedIdentityClientId = "fa7f8761-f449-4336-ace2-b83845b47245"
};
var client = new SecretClient(new Uri("https://kv-demo.vault.azure.net/"), new DefaultAzureCredential(credentialOptions));
KeyVaultSecret secret = client.GetSecret("SecretName");
```

Możemy też zrobić kompletny mix i mieć zarówno System Assigned Identity jak i kilka User Assigned które do tego są przypisane wielu zasobom. Wtedy ponownie, z poziomu kodu decydujemy którego Identity chcemy użyć.

Inny jest też cykl życia takiego Identity. Tym razem nie jest on powiązany z cyklem życia zasobu. User Identity możemy stworzyć zanim powstanie maszyna wirtualna, możemy już nadać porządane role, następnie stworzyć maszynę i przypisać Identity. Kiedy z jakiegoś powodu maszynę kasujemy, wcale nie musimy usuwać Identity. Być może maszyna zaraz powróci a Identity będzie już czekało z odpowiednimi uprawnieniami. Może to być przydatne w sytuacjach, w których chcemy nadać uprawnienia zanim powstanie zasób lub jeśli tworzymy zasoby zależne od siebie. Na przykład, w jednym Bicep template chcesz:
- stworzyć App Service
- stworzyć Key Vault 
- nadać uprawnienia do odczytywania sekretów z Key Vaulta przez App Service
- ustawić URL Key Vaulta w App Service

Korzystając z System Assigned Managed Identity jest to trudne, ponieważ:
- Aby nadać uprawnienia do Key Vaulta dla System Assigned Managed Identity, nasz App Service musi istnieć. A więc App Service Musi powstać pierwszy.
- Jeśli App Service powstaje pierwszy to jeszcze nie ma Key Vaulta, a więc nie możemy przekazać jego URL do App Service i koło się zamyka.

Można sobie poradzić z tą sytuacją korzystając z User Assigned Managed Identity. Wtedy proces wygląda tak:
- Tworzy się User Assigned Managed Identity.
- Powstaje Key Vault i od razu można nadać w nim uprawnienia dla uprzednio stworzonego Identity.
- Powstaje App Service do którego przekazujemy URL Key Vaulta oraz przpinamy istniejące już Identity

Podsumowując, User Assigned Identity:
- Ma cykl życia niepowiązany z zasobami które go używają. To my decydujemy kiedy Identity powstaje i kiedy jest usuwane.
- Może być przypisane do wielu zasobów, oraz jeden zasób może korzystać z wielu Identity.
- Może mieć nadane role zanim zostanie przypisane do konkretnego zasobu.
- Może współistnieć z System Assigned Managed Identity.

## Wybór rodzaju Managed Identity

Z mojego doświadczenia zazwyczaj wygodniej jest używać User Assigned Identity. U mnie wynika to przede wszystkim z chęci dzielenia jednego identity np. przez różne deployment sloty w App Service. Drugim powodem jest chęć nadania uprawnień zanim powstaną zasoby. Jest to szczególnie wygodne kiedy wiele rzeczy powstaje na raz w jednym Bicep template, lub jeśli jeszcze nie ma potrzeby tworzyć VMki czy App Serviceu i za nie płacić, a można już przygotować uprawnienia które będą wymagane. Wtedy identity można stworzyć, nadać role, a kiedy już powstaną serwery można tylko przypisać Identity i sprawa załatwiona.

Trzeba jednak pamiętać o konieczności manualnego zarządzania cyklem życia i uważać żeby nie zostały nam gdzieś Identity których nikt nie używa.

W większości przypadków sugerowałbym wybranie User Assigned Managed Identity, ponieważ moim zdaniem jest mimo wszystko wygodniejsze w użyciu, oraz oferuje więcej możliwości. W prostych przypadkach gdzie na pewno nie wykorzystamy możliwości User Assigned można spokojnie wybrać System Assigned. 

Jakąkolwiek decyzję podejmiemy, jest ona łatwo zmienialna, także nie trzeba się nad tym jakoś specjalnie głowić i wybrać to, co w danej chwili jest dla nas wygodniejsze.