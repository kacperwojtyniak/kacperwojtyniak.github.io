# Cykl życia blobów sposobem na ograniczenie kosztów

Niedawno Microsoft dodał [możliwość śledzenia](https://azure.microsoft.com/en-gb/updates/azure-blob-storage-last-access-time-tracking-now-generally-available/) kiedy blob został ostatnio użyty, tj. kiedy ktoś go np. pobrał, miał do niego dostęp. Ma to pozwolić na lepsze zarządzanie cyklem życia blobów, a tym samym ma pomóc w zoptymalizowaniu kosztów. Jest to dobra okazja żeby napisać kilka słów na temat owego cyklu i jak możemy nim automatycznie zarządzać.

## Access tier

Po polsku można to nazwać "warstwa dostępu" ale wydaje mi się, że nie brzmi to dobrze i nie oddaje dobrze o czym mówimy. Pozwolę więc sobie używać angielskiego access tier.

Bloby mogą być umieszczone w jednym z trzech access tier:
- hot
- cool
- archive

Różnią się one czasem dostępu do plików, a także kosztami. W przypadku hot, mamy najszybszy dostęp do plików, najniższe koszty związane z operacjami na plikach, natomiast **najwyższe koszty składowania danych**. Schodząc do poziomu cool, koszt składowania danych jest mniejszy, za to płacimy więcej za operacje. W przypadku archive, pliki są przechowywane offline i żeby się do nich dostać musimy najpierw zmienić tier na cool lub hot. To potrafi długo trwać, sporo kosztuje, za to składowanie danych jest najtańsze. To tak w skrócie, po szczegóły odsyłam na [strony Microsoftu](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-storage-tiers).

## Automatyczne przenoszenie plików między hot-cool-archive

Pliki których używamy często będziemy zazwyczaj trzymać w tierze hot, te do których zaglądamy rzadko przeniesiemy do cool a pliki które z jakiegoś powodu musimy przetrzymywać ale z nich nie korzystamy mogą leżeć w archive. Takie przenoszenie można oczywiście robić ręcznie, można napisać jakiś skrypt który to zrobi. Można również posłużyć się automatem wbudowanym w Azure, tj. [Lifecycle management](https://docs.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview).

Automatyczne przenoszenie pomiędzy tierami może być wykonywane na podstawie ilości dni które upłynęły od ostatniej modyfikacji bloba, oraz od niedawna na podstawie czasu od ostatniego dostępu do bloba.

### Czas modyfikacji

Możemy zdecydować, że wszystkie bloby w storage account, które nie były modyfikowane od 30 dni powinny być przeniesione z hot do cool. W portalu Azure, w storage account wystarczy wybrać zakładkę `Lifecycle management`, nadać nazwę reguły, wybrać do jakich blobów reguła ma się stosować, oraz wpisać po ilu dniach blob ma zostać przeniesiony. Możemy bloba przenieść do cool, archive, lub go usunąć.

[![regula](/images/cykl-zycia-blobow/1.png)](/images/cykl-zycia-blobow/1.png)

[![parametry](/images/cykl-zycia-blobow/2.png)](/images/cykl-zycia-blobow/2.png)

### Czas od ostatniego dostępu

Niedawno dostaliśmy opcję przenoszenia blobów pomiędzy tierami na podstawie nowego property. Jest nim `lastAccessedOn`. Tego property nie zobaczycie w portalu, natomiast można go użyć w polityce cyklu życia, oraz zobaczyć np. poprzez az cli.

Żeby jednak to zrobić należy włączyć śledzenie czasu dostępu. Można to zrobić w zakładce `Lifecycle management` w portalu, lub poprzez cli czy też template.

[![sledzenie_czasu_dostepu](/images/cykl-zycia-blobow/3.png)](/images/cykl-zycia-blobow/3.png)

Od tego czasu korzystając komendy `az storage blob show` zobaczymy wypełnione property `lastAccessedOn`.

[![cli](/images/cykl-zycia-blobow/4.png)](/images/cykl-zycia-blobow/4.png)

Teraz możemy również skorzystać z tego przy ustawianiu polityk.

[![polityka](/images/cykl-zycia-blobow/5.png)](/images/cykl-zycia-blobow/5.png)

Warto zwrócić uwagę na dodatkową opcję, której nie ma w przypadku polityki opartej o czas modyfikacji. Mamy do dyspozycji `Move to cool storage and move back if accessed`. Ustawiając politykę na 30 dni, blob zostanie przeniesiony po 30 dniach od ostatniego dostępu do pliku do cool tier. Natomiast jeśli nastąpi dostęp do pliku, zostanie on znowu przeniesiony do tiera hot, z założeniem, że będzie częściej używany. Oczywiście nie musimy wybierać tej opcji, ciągle jest dostępne standardowe `Move to cool storage`.

## Oszczędności

Jak duże oszczędności można w ten sposób wygenerować będzie zależało od tego ile macie danych. Im więcej danych, tym więcej oszczędności. Rzućmy okiem na [cennik](https://azure.microsoft.com/en-gb/pricing/details/storage/blobs/).

[![cennik](/images/cykl-zycia-blobow/6.png)](/images/cykl-zycia-blobow/6.png)

Do poziomu 50 TB, koszt przechowywania 1GB danych w tierze hot, jest 80% większy niż w cool. Archive jest już w ogóle poza konkurencją. 

Różnica w kosztach operacji też jest znacząca. Tutaj najtańszy jest hot.

[![operacje](/images/cykl-zycia-blobow/7.png)](/images/cykl-zycia-blobow/7.png)

## Podsumowanie

Jeśli masz dużo danych których nie używasz albo używasz rzadko, zdecydowanie warto się zainteresować ustawieniem polityk i wybrać odpowiedni access tier. Należy przy tym pamiętać o dwóch rodzajach kosztów - za składowanie danych oraz operacje. Nie zawsze cool będzie tańszy niż hot.

## Bonus 

Polityki są też dobrym narzędziem do usuwania starych plików. Kiedyś chciałem usunąć setki gigabajtów danych, ale nie mogłem usunąć całego kontenera, tylko niektóre, stare już pliki. Takie kasowanie z pomocą storage explorera trwało wieczność. Z pomocą przyszła polityka cyklu życia. Ustawiłem jak stare pliki mają zostać usunięte i reszta zadziała się sama. Przydatna była też opcja filtrowania plików po prefixie. Możemy ustawić nie tylko ilość dni od ostatniej modyfikacji czy ostatniego dostępu, ale także prefix nazwy pliku, np. możemy zastosować politykę do plików zaczynających się od `logs/old-nonexistent-app`. Do filtrowania mogą posłużyć także [index tagi](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-manage-find-blobs?tabs=azure-portal).