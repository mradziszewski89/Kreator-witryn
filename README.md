# SharePoint Provisioning Framework

[![PowerShell 5.1](https://img.shields.io/badge/PowerShell-5.1-5391FE?logo=powershell&logoColor=white)](#wymagania)
[![SharePoint SE](https://img.shields.io/badge/SharePoint-Subscription%20Edition-0078D4)](#co-robi-rozwiazanie)
[![Tryby pracy](https://img.shields.io/badge/Tryby-DryRun%20%7C%20Execute-1B3A6B)](#szybki-start)
[![Wiki](https://img.shields.io/badge/Wiki-Ready%20for%20GitHub-107C10)](wiki/Home.md)

Framework automatyzuje provisioning środowisk i struktur informacji w Microsoft SharePoint Subscription Edition (on-premise) na podstawie konfiguracji JSON wygenerowanej lokalnie z formularza HTML. Rozwiązanie łączy formularz administracyjny, walidację, provisioning PowerShell, raport HTML, log tekstowy i generator dokumentacji DOCX.

> Repozytorium jest przeznaczone dla administratorów SharePoint pracujących w środowiskach on-premise. Główny skrypt zakłada uruchomienie z uprawnieniami administratora lokalnego i Farm Administrator.

## Co robi rozwiązanie

| Obszar | Zakres |
| --- | --- |
| Formularz wejściowy | Lokalny formularz `SP-Provisioning-Form.html` budujący konfigurację JSON bez backendu |
| Provisioning SharePoint | Tworzenie Web Application, Content Database, kolekcji witryn i podwitryn |
| Uprawnienia | Dziedziczenie, łamanie dziedziczenia, mapowanie ról i przypisania do użytkowników oraz grup |
| Artefakty witryn | Automatyczne tworzenie list i bibliotek dokumentów |
| Bezpieczeństwo operacyjne | Tryb `DryRun`, walidacja wstępna, obsługa błędów i rollback |
| Raportowanie | Log tekstowy + samowystarczalny raport HTML z podsumowaniem sesji |
| Dokumentacja | Generator `SP-Framework-Dokumentacja.docx` bez zależności od Microsoft Word |

## Architektura w skrócie

```mermaid
flowchart LR
    A[Administrator] --> B[SP-Provisioning-Form.html]
    B --> C[config.json]
    C --> D[Start-SPProvisioning.ps1]
    D --> E[Walidacja]
    E --> F[Web Application]
    F --> G[Content Database]
    G --> H[Site Collections]
    H --> I[Subsites]
    I --> J[Permissions i Lists]
    J --> K[Raport HTML i log]
```

## Najważniejsze wyróżniki

- Jeden punkt wejścia dla administratora: formularz HTML do budowy konfiguracji i skrypt PowerShell do wykonania provisioningu.
- Jasny podział odpowiedzialności na moduły PowerShell w katalogu `Modules/`.
- Obsługa `DryRun`, która pozwala ocenić zakres zmian przed wykonaniem operacji w farmie.
- Walidacja reguł SharePoint, w tym `managedPath`, unikalności URL-i, poprawności nazw baz i ograniczenia "jedna baza tylko dla Site Collection".
- Raport HTML generowany lokalnie, bez zależności od Internetu i bez zewnętrznych bibliotek.
- Dodatkowy generator pełnej dokumentacji DOCX oparty o Open XML.

## Struktura repozytorium

```text
SharePoint-Provisioning/
|-- SP-Provisioning-Form.html
|-- Start-SPProvisioning.ps1
|-- config-example.json
|-- Generate-Documentation.ps1
|-- Run-GenerateDocumentation.ps1
|-- README-Architektura.md
|-- SP-Framework-Dokumentacja.docx
|-- Modules/
|   |-- SP-Logging.psm1
|   |-- SP-Validation.psm1
|   |-- SP-WebApplication.psm1
|   |-- SP-ContentDatabase.psm1
|   |-- SP-SiteCollection.psm1
|   |-- SP-Subsites.psm1
|   |-- SP-Permissions.psm1
|   |-- SP-Lists.psm1
|   `-- SP-Reporting.psm1
`-- wiki/
    |-- Home.md
    |-- Dokumentacja-techniczna.md
    |-- Uruchomienie-i-diagnostyka.md
    `-- _Sidebar.md
```

## Szybki start

1. Otwórz `SP-Provisioning-Form.html` w przeglądarce i przygotuj konfigurację.
2. Wyeksportuj plik JSON.
3. Uruchom `Start-SPProvisioning.ps1` najpierw w trybie `DryRun`.
4. Zweryfikuj raport HTML i log.
5. Uruchom provisioning w trybie `Execute`.

```powershell
# 1. Podgląd bez zmian w farmie
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode DryRun

# 2. Wykonanie właściwe
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode Execute

# 3. Wykonanie bez dodatkowego potwierdzenia
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode Execute -Force
```

## Jak przebiega provisioning

1. Skrypt ładuje moduły i inicjalizuje logowanie.
2. Odczytuje oraz parsuje konfigurację JSON do `hashtable`.
3. Waliduje konfigurację, środowisko, połączenie z SQL i podstawowe kolizje SharePoint.
4. Tworzy Web Application i Application Pool.
5. Tworzy lub dobiera Content Database dla każdej Site Collection.
6. Tworzy kolekcje witryn, podwitryny, uprawnienia oraz listy.
7. Buduje raport HTML i zamyka sesję logowania.

## Model konfiguracji

Rozwiązanie pracuje na jednym pliku JSON zawierającym sekcje:

- `metadata`
- `webApplication`
- `applicationPool`
- `sqlServer`
- `siteCollections`

<details>
<summary>Przykładowy fragment konfiguracji</summary>

```json
{
  "webApplication": {
    "name": "SharePoint - Intranet 80",
    "url": "http://intranet.firma.local",
    "port": 80
  },
  "applicationPool": {
    "name": "SharePoint - Intranet AppPool",
    "mode": "new",
    "account": "FIRMA\\sp_apppool",
    "passwordSecured": "[SECURED]"
  },
  "sqlServer": {
    "server": "SQL01",
    "contentDatabase": "WSS_Content_Intranet"
  },
  "siteCollections": [
    {
      "title": "Portal Intranet",
      "managedPath": "/",
      "url": "/",
      "template": "CMSPUBLISHING#0"
    }
  ]
}
```

</details>

## Wymagania

| Obszar | Wymaganie |
| --- | --- |
| System | Windows Server 2019/2022 |
| PowerShell | Windows PowerShell 5.1 |
| SharePoint | SharePoint Subscription Edition |
| Shell | SharePoint Management Shell lub dostęp do `Microsoft.SharePoint.PowerShell` |
| Uprawnienia | Administrator lokalny + Farm Administrator |
| SQL Server | Dostęp sieciowy i uprawnienia do tworzenia / podpinania Content Databases |

## Istotne założenia

- Podwitryny nie mogą mieć osobnych Content Databases. Rozwiązanie respektuje to ograniczenie SharePoint i zgłasza ostrzeżenie przy błędnej konfiguracji.
- Dla `applicationPool.mode = "new"` konto musi być wcześniej zarejestrowane jako Managed Account w farmie. Pole `passwordSecured` w JSON jest wyłącznie znacznikiem z formularza.
- `DryRun` loguje planowane działania, ale nie wykonuje zmian w farmie i pomija część testów środowiskowych zależnych od trybu pełnego.
- Raport HTML i log tekstowy są zapisywane domyślnie w katalogu `Logs/`.

## Dokumentacja

- [Wiki Home](wiki/Home.md)
- [Dokumentacja techniczna](wiki/Dokumentacja-techniczna.md)
- [Uruchomienie i diagnostyka](wiki/Uruchomienie-i-diagnostyka.md)
- [Opis architektury](README-Architektura.md)
- `SP-Framework-Dokumentacja.docx`

## Materiały dodatkowe

- `config-example.json` zawiera gotowy przykład wielopoziomowej struktury.
- `Run-GenerateDocumentation.ps1` uruchamia generator DOCX z wymuszonym UTF-8 BOM, co upraszcza pracę w PowerShell 5.1.
- `Generate-Documentation.ps1` generuje dokumentację wdrożeniową i techniczną bez użycia Microsoft Word.

## Dla maintainerów

Jeśli publikujesz projekt w GitHub, folder `wiki/` jest przygotowany jako pakiet startowy do repozytorium Wiki. Plik `Home.md` pełni rolę strony głównej, a `_Sidebar.md` dostarcza boczną nawigację.
