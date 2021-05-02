# Uwierzytelnianie no-code w Azure

Czy wiesz, że w Azure masz możliwość dodania do swojej web aplikacji uwierzytelniania bez napisania nawet jednej linijki kodu? Pozwól, że zaprezentuję.

Do przykładu użyję Azure Functions bo mam wrażenie, że to właśnie tutaj omawiane rozwiązanie przyda się najbardziej. Chciałbym jednak zaznaczyć, że zadziała to tak samo dla App Service.

## Poziomy autoryzacji dla funkcji HTTP

Tworząc funkcję z triggerem HTTP masz do wyboru trzy poziomy autoryzacji:
- Anonymous - funkcja jest dostępna dla każdego bez żadnych kluczy, tokenów itp.
- Function - żeby wywołać funkcję w żądaniu musimy zawrzeć klucz wygenerowany dla konkretnej funkcji
- Admin - podobnie jak w przypadku poprzedniej opcji, musimy użyć klucza, tym razem tzw. master key

Te opcje działają, ale nie są najwygodniejsze. Kiedy chcemy żeby nasza funkcja była dostępna publicznie to okej, ustawiamy Anonymous i wszystko gra. Gorzej, kiedy potrzebujemy naszą funkcję zabezpieczyć. Opierając nasze bezpieczeństwo na kluczach mamy kilka problemów:
- klucz trzeba wygenerować
- trzeba go gdzieś bezpiecznie przechowywać
- jeśli inny zespół będzie się komunikował z naszą funkcją, trzeba klucz do tego zespołu przekazać
- powinniśmy pomyśleć o rotacji kluczy
- etc.

Po co to wszystko kiedy mamy OAuth?

## Włączamy uwierzytelnianie

W Azure Portal należy znaleźć aplikację którą chcemy zabezpieczyć i wybrać zakładkę `Authentication`. Następnie kliknąć `Add identity provider`.

[![krok_1](/images/uwierzytelnianie-no-code/1.png)](/images/uwierzytelnianie-no-code/1.png)

### Wybór dostawcy tożsamości

Mamy tutaj kilka opcji i nie musimy się ograniczać do Azure. Ja jednak wybieram Azure Active Directory. Dzięki temu większość konfiguracji będzie ogarnięta automatycznie, a w dodatku będę mógł później wykorzystać [Managed Identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview). 

[![krok_2](/images/uwierzytelnianie-no-code/2.png)](/images/uwierzytelnianie-no-code/2.png)

### Konfiguracja aplikacji

W kolejnym kroku wybieramy czy chcemy stworzyć nową rejestrację w Azure AD, wybrać istniejącą, czy będziemy wspierać jednego tenanta czy wielu oraz jaką odpowiedź zwrócić dla anonimowych żądań. W tym przykładzie wszystkie opcje zostawiam domyślne, oprócz odpowiedzi dla anonimowych żądań. Tutaj wybieram `401 Unauthorized` ponieważ w kolejnym wpisie chcę Ci pokazać jak wykorzystać ten mechanizm do komunikacji API-API. Ciebie zachęcam do wypróbowania opcji z odpowiedzią `302 Found Redirect`. Sprawdź co się stanie.

[![krok_3](/images/uwierzytelnianie-no-code/3.png)](/images/uwierzytelnianie-no-code/3.png)

### Uprawnienia

Ostatni krok to wybranie uprawnień. Myślę, że temat już Ci znany. Domyślnie wybrany jest jeden "scope" i jest to `User.Read`. Możemy też dodać kolejne jak np. `email` co pozwoli naszej aplikacji na dostęp do adresu email użytkownika.

[![krok_4](/images/uwierzytelnianie-no-code/4.png)](/images/uwierzytelnianie-no-code/4.png)

### Bearer token wymagany

Po zakończeniu procedury wszystkie żądania wymagać będą prawidłowego tokena Bearer. Każdy kto chce się dostać do naszej funkcji, musi się uwierzytelnić w naszym Azure AD.

Mimo iż moja funkcja ma ustawiony "Authorization level" na `Anonymous`, po przejściu powyższych kroków nie jest już dostępna dla nieuwierzytelnionych użytkowników. Poniżej widzisz dwa żądania, jedno przed włączeniem uwierzytelniania, jedno po. Jak widać w obu przypadkach nie podaję żadnych kluczy czy tokenów. Przed włączeniem uwierzytelniania otrzymuję status `200 OK`, po jest to `401 Unauthorized`.

[![krok_5](/images/uwierzytelnianie-no-code/5.png)](/images/uwierzytelnianie-no-code/5.png)

## Podsumowanie

W kilku prostych krokach możemy całkiem nieźle zabezpieczyć nasze aplikacje webowe. Może to być Azure Functions lub Azure AppService. Co ciekawsze, nie musimy w ogóle modyfikować naszego kodu. Należy jednak pamiętać, że jest to tylko warstwa uwierzytelniania, bez kontroli uprawnień. Ale spokojnie, uprawnienia też możemy łatwo ogarnąć. To już będzie wymagało zmian w kodzie, ale wciąż będzie to relatywnie łatwe.

W kolejnym wpisie przeczytasz jak z poziomu innej aplikacji w Azure wykonać zapytanie do funkcji którą właśnie zabezpieczyliśmy. Do tego pokażę Ci jak dołożyć do tego uprawnienia. Najlepsze jest to, że w dalszym Ciągu nie będziemy musieli zarządzać żadnymi kluczami czy innymi sekretami.