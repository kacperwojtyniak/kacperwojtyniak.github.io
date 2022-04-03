# Usługi PaaS, sieci i Private Endpoint

Jeśli pracowałeś kiedyś, lub pracujesz nadal, ze środowiskami tak zwanymi on-premises, na pewno spotkałeś się z tym, że wszystko jest za Firewallem. Z reguły nie ma możliwości dostania się do bazy danych z publicznego internetu. Dyski czy aplikacje wewnętrzne dostępne są tylko z użyciem VPNa. Generalnie wszystko co nie jest kierowane do klienta, jest wycięte z publicznego internetu w celu zapewnienia bezpieczeństwa.

W chmurze nie zawsze tak jest. Często wszystko co tworzymy, jest od razu wystawione do świata. Czasami w pełni, czasami częściowo. Oczywiście wszystko wymaga uwierzytelnienia czy to hasłem czy naszą tożsamością. Nie zmienia to jednak faktu, że nawiązać połączenie można.

W tym wpisie chciałbym Ci pokazać w jaki sposób można ograniczyć dostęp do usług typu PaaS. Wykorzystam do tego [Private Endpoint](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview). Przykład będzie oparty o Azure SQL, ale wygląda to tak samo dla wszystkich usług.

## Punkt wyjścia

W przypadku Azure SQL, już domyślnie mamy pewien poziom zabezpieczeń. Mamy tutaj Firewalla i aby móc się podpiąć do bazy, należy dodać adres IP do listy zaufanych. Gorzej jeśli do bazy łączymy się np z Azure App Service albo Azure Function. Nie mamy wtedy stałego adresu który moglibyśmy dodać do Firewalla. W związku z tym włączamy opcję "Allow Azure services and resources to access this server" i sprawa załatwiona. 

[![default](/images/uslugi-paas-sieci-i-private-endpoint/default.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/default.jpg)

Mogłoby się wydawać, że jest to dobra konfiguracja. Dostęp do bazy jest za Firewallem, więc nikt niepożądany się nie dostanie. Jednocześnie umożliwiamy dostęp tylko z Azura, więc nasze aplikacje nie mają problemu. 

Włączając tą opcję, umożliwiamy dostęp z całego Azura. Nie tylko z naszej subskrypcji czy tenanta. **Każdy, w dowolnej subskrypcji, może połączyć się z naszą bazą. Może do tego wykorzystać dowolny zasób azurowy, tj. web apkę, azure function, maszynę wirtualną, etc.**

Oczywiście serwer zabezpieczony jest jeszcze hasłem, ale mimo wszystko zwiększa to obszar ataku.

W przypadku innych usług, np. Storage Account domyślnie każdy może próbować dobrać się do naszych plików. Ponownie, jest to wszystko zabezpieczone kluczami, lub poprzez IAM, jednak nie ma zabezpieczenia na poziomie sieciowym. Możemy to zmienić i tutaj też dodać tylko konkretne adresy IP do listy zaufanych, lub konkretne sieci, natomiast powróci problem zapewnienia dostępu dla usług które nie mają stałego adresu IP.

## Private endpoint

Na ratunek przychodzi [Private Endpoint](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview). W skrócie, Private Endpoint umożliwia stworzenie stałego, prywatnego adresu IP dla usługi którą chcemy zabezpieczyć w naszej sieci wirtualnej.

Dzięki temu możemy całkowicie zablokować dostęp do zasobu, z wyjątkiem dostępu przez ten jeden, prywatny adres IP. Adres ten jest w naszej sieci wirtualnej, a to my decydujemy które aplikacje zintegrujemy z tą siecią. W ten sposób mamy pełną kontrolę nad tym, kto jest w stanie uzyskać dostęp do bazy, storage accounta, itd.

Dodatkowym atutem jest to, że ruch nie przechodzi już przez publiczny internet. Jeśli App Service uzyskuje dostęp do bazy, bez Private Endpointa ruch ten przejdzie przez publiczną sieć. Kiedy użyjemy Private Endpointa wszystko jest zamknięte w prywatnej sieci i nic nie przechodzi przez tak zwany "świat".

Najlepiej będzie to zrozumieć na przykładzie.

## Przykład - Azure SQL

Przypomnę jeszcze raz punkt wyjścia.

[![default](/images/uslugi-paas-sieci-i-private-endpoint/default.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/default.jpg)

W takiej konfiguracji dostęp do serwera dostępny jest:
- z publicznego internetu tylko dla adresów IP dodanych do listy - obecnie lista jest pusta, więc nikt się nie dostanie
- z sieci Azura - każdy może się podłączyć do bazy jeśli tylko jest w Azure, tj. dowolna maszyna wirtualne, app service, itd.

**Cel**: Dostęp do bazy danych możliwy tylko z wewnątrz sieci wirtualnej.

### Przygotowanie środowiska
Jeśli chcesz podążać za przykładem, identyczny punkt wyjścia możesz uzyskać wykonując poniższy fragment powershella i Azure CLI.

```pwsh
# Nazwa resource grupy
$rg = 'rg-pe-demo'
# Lokalizacja
$l = 'swedencentral'
# Nazwa serwera SQL
$server = 'sql-pe-demo'
# Nazwa bazy danych
$db = 'sqldb-pe-demo'
# Nazwa użytkownika administratora serwera
$u = 'kacper'
# Hasło administratora
$p = '<wpisz-super-silne-haslo!>'

# Stworzenie resource grupy
az group create -n $rg -l $l

# Stworzenie SQL Serwera
az sql server create -n $server --location $l -g $rg -u $u -p $p

# Włączenie "Allow access from azure services"
az sql server firewall-rule create -g $rg -s $server -n azureservices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0

# Stworzenie bazy danych
az sql db create -n $db -g $rg -s $server --sample-name AdventureWorksLT

```

Aby testować połączenie do bazy, najłatwiej będzie posłużyć się maszyną wirtualną i Sql Server Management Studio. Możesz stworzyć taką maszynę z zainstalowanym SSMS oraz otwartym portem RDP wykonując poniższy skrypt.

```pwsh
$sqlImage = "MicrosoftSQLServer:sql2019-ws2019:standard:15.0.211109"
$vnetName = 'vnet-pe-demo'
az vm create `
--name 'vm-pe-demo' `
--resource-group $rg `
--admin-username $u `
--admin-password '<wpisz-super-silne-haslo!>' `
--image $sqlImage `
--size Standard_D2s_v4 `
--vnet-name $vnetName `
--subnet 'subnet-vm-pe-demo' `
--subnet-address-prefix "10.0.1.0/24" `
--location $l `
--nsg-rule RDP
```

Spróbuj połączyć się z maszynką przez RDP, otwórz Sql Server Management Studio i spróbuj połączyć się do bazy. Powinieneś się dostać bez problemu. Mimo, że nie dodaliśmy adresu IP tej maszyny do Firewalla SQLa. Ze swojego komputera już się nie podłączysz.

### Blokada dostępu

Czas zablokować dostęp do bazy kompletnie.

```pwsh
# Wyłączenie dostępu z publicznego internetu
az sql server update -n $server -g $rg --set publicNetworkAccess="Disabled"
```

Od teraz nikt nie powinien się przebić przez Firewalla. Sprawdź, próbując połączyć się z poprzednio stworzonej maszyny wirtualnej.

### Dodanie Private Endpoint dla SQL Servera

Private Endpoint umieszczany jest w sieci wirtualnej. Sieć stworzyliśmy już podczas tworzenia maszyny wirtualnej. Teraz potrzebujemy dodać tylko podsieć.

```pwsh
# Nazwa podsieci
$subnet = 'subnet-pe'
# Stworzenie podsieci w uprzednio stworzonej sieci
az network vnet subnet create `
    --address-prefixes 10.0.2.0/24 `
    --name $subnet `
    -g $rg `
    --vnet-name $vnetName `
    --disable-private-endpoint-network-policies true
```

Następnie potrzebne jest Resource ID podsieci i SQL Servera oraz private link resource group id. 

```pwsh
$resourceId = az sql server show -n $server -g $rg --query 'id'
$groupId = az network private-link-resource list --id $resourceId --query '[0].properties.groupId'
$subnetId = az network vnet subnet show -n $subnet -g $rg --vnet-name $vnetName --query 'id'
```

Teraz możemy stworzyć Private Endpoint dla SQL Servera.

```pwsh
# Stworzenie Private Endpointa
az network private-endpoint create `
    --name $peName `
    --resource-group $rg `
    --subnet $subnetId `
    --private-connection-resource-id $resourceId `
    --group-id $groupId `
    --connection-name 'st-pe' `
    --location $l
```

W resource grupie powinieneś zobaczyć Private Endpoint oraz jego Network Interface.

[![private-endpoint-resources](/images/uslugi-paas-sieci-i-private-endpoint/private-endpoint-resources.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/private-endpoint-resources.jpg)

Jak otworzysz Network Interface to zobaczysz, że ma on nadany prywatny adres IP, pochodzący z naszego VNETa.
[![private-ip](/images/uslugi-paas-sieci-i-private-endpoint/private-ip.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/private-ip.jpg)

### DNSy

Na tym etapie mamy już uruchomiony dostęp jedynie przy pomocy adresu prywatnego z VNETa. Jednak jeśli spróbujesz się teraz połączyć z maszyny wirtualnej do bazy danych, dalej nie będzie to możliwe. A przynajmniej nie tak od razu. Co prawda maszyna znajduje się w tej samej sieci, więc powinna być w stanie dobić się do prywatnego adresu bazy, ale brakuje DNSów. 

Próbując podłączyć się do `sql-pe-demo.database.windows.net` domena `database.windows.net` nadal wskazuje na publiczny adres IP SQL Servera. We wcześniejszych krokach zablokowaliśmy tam dostęp dla wszystkich. 

Aby uzyskać dostęp do bazy za pośrednictwem Private Endpointa należy przestawić DNSy tak, aby `database.windows.net` wskazywało na `privatelink.database.windows.net`. Jeśli korzystasz z własnych serwerów DNS możesz to zrobić tam. Ba, możesz nawet na samej maszynie w pliku `hosts` ustawić prywatny adres IP bazy `10.0.2.4 sql-pe-demo.database.windows.net` - spróbuj!

Zamiast tego, ja posłużę się Azurowym Private DNS Zone.

Najpierw tworzę Private DNS Zone.

```pwsh
$dnsZoneName = 'privatelink.database.windows.net'
az network private-dns zone create `
--resource-group $rg `
--name $dnsZoneName
```

W resource grupie pojawi się nowy zasób.

[![private-zone](/images/uslugi-paas-sieci-i-private-endpoint/private-zone.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/private-zone.jpg)

Lecz jak w niego wejdziesz to zobaczysz, że nie ma jeszcze żadnych rekordów.

[![empty-zone](/images/uslugi-paas-sieci-i-private-endpoint/empty-zone.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/empty-zone.jpg)

Następnie należy podpiąć zone do VNETa. Od tego momentu zasoby zintegrowane z tym VNETem, będą korzystały rekordów z prywatnej zony, czy też strefy.

```pwsh
# Podpięcie Private DNS do VNETa
az network private-dns link vnet create `
    --resource-group $rg `
    --zone-name $dnsZoneName `
    --name 'dns-link-pe' `
    --virtual-network $vnetName `
    --registration-enabled false
```

Na sam koniec wystarczy dodać rekord A, wskazujący na prywatny adres IP.

```pwsh
az network private-endpoint dns-zone-group create `
--resource-group $rg `
--endpoint-name $peName `
--name 'pe-zone-group' `
--private-dns-zone $dnsZoneName `
--zone-name 'pezone'
```

Jak widać, stworzył się rekord A wskazujący na prywatny IP 10.0.2.4

[![record-a](/images/uslugi-paas-sieci-i-private-endpoint/record-a.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/record-a.jpg)

Od teraz wszystkie zasoby będące w tej sieci, próbując się połączyć z `sql-pe-demo.database.windows.net` będą kierowane na prywatny adres IP. Wszyscy poza VNETem odbiją się od ściany.

## Połączenie z innych PaaS-ów

Możliwe, że zastanawiasz się co z dostępem z poziomu App Serwisu lub innych rozwiązań PaaS. W przykładzie wyżej mogłeś sprawdzić, że mimo zablokowania dostępu do bazy, da się do niej dostać z wewnątrz VNETa. No dobra, to oczywiste że maszyna wirtualna potrzebuje VNET do działania, a co z rozwiązaniami PaaS?

W App Service można włączyć integrację z VNETem. Wtedy tak samo będzie miał dostęp do Private Endpointa. Tak samo potrzebny jest Private DNS Zone.

Włączyć integrację można z poziomu portalu, lub skryptami czy też szablonami Bicep, ARM, itd.

[![app-service-vnet](/images/uslugi-paas-sieci-i-private-endpoint/app-service-vnet.jpg)](/images/uslugi-paas-sieci-i-private-endpoint/app-service-vnet.jpg)

Będąc na zakładce Networking możesz zauważyć że jest tutaj również opcja Private Endpoint. Zgadza się, w taki sam sposób możemy ograniczyć dostęp do App Serwisu. Jeśli mamy jakieś API które jest wykorzystywane tylko przez nasze inne aplikacje i nie powinno być dostępne z publicznego internetu, możemy ograniczyć dostęp do konkretnego VNETa, tak jak w przykładzie z SQLem.

### Azure Functions

Jeśli rozważasz ograniczenie dostępu do zasobów poprzez użycie Private Endpoint i korzystasz Azure Functions, musisz wiedzieć, że niestety ale plan Serverless nie jest w stanie integrować się z VNETami. Przez to, musisz zrezygnować z Serverless i przejść na Premium, lub na Dedicated plan.

## VNET Peering

Do Private Endpointa mamy dostęp z wewnątrz VNETa. To już oczywiste. Mamy też dostęp z sieci sparowanych, tak zwany VNET peering. Jeśli VNET który utworzyliśmy w przykładzie wyżej sparujemy z drugim VNETem, to wszystko do się znajduje w sparowanym VNEcie będzie miało dostęp do Private Endpointa. Jedyne co trzeba dodatkowo zrobić to podpiąć Private DNS Zone do parowanego VNETa. Inaczej DNSy nie wskażą na adres prywatny.

## Podsumowanie

Private Endpoint to dosyć proste w konfiguracji narzędzie które umożliwia ograniczenie dostępu do usług PaaS co pozwala na zwiększenie bezpieczeństwa. Jeśli coś nie musi być wystawione do publicznego internetu, raczej nie powinno być. Niestety często dzieje się tak, że chmurowe rozwiązania pozostawiamy otwarte. Same hasła czy inne klucze mogą nie być wystarczające. Jeśli tylko jest możliwość, lepiej ucinać ruch już na poziomie sieci.

Niestety nie zawsze będziemy mogli wybrać takie rozwiązanie. Jeśli zależy nam na wykorzystaniu technologii serverless, gdzie nie mamy możliwości intergracji z VNETem, nie będziemy w stanie skorzystać z Private Endpointa.

## Linki
- [Private Endpoint Documentation](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [John Savill Private Link Deep Dive](https://www.youtube.com/watch?v=57ZwdztCx2w)
- [App Service VNET Integration](https://docs.microsoft.com/en-us/azure/app-service/overview-vnet-integration)