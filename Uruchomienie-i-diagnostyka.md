# Uruchomienie i diagnostyka

## Wymagania operacyjne

| Obszar | Minimalne wymaganie |
| --- | --- |
| System | Windows Server z dostępem do farmy SharePoint |
| PowerShell | Windows PowerShell 5.1 |
| SharePoint | SharePoint Subscription Edition |
| Uprawnienia | Administrator lokalny oraz Farm Administrator |
| SQL | Dostęp do serwera i możliwość podpięcia / utworzenia Content Database |
| Shell | SharePoint Management Shell albo poprawnie załadowany `Microsoft.SharePoint.PowerShell` |

## Zalecana kolejność pracy

1. Przygotuj konfigurację w `SP-Provisioning-Form.html`.
2. Wyeksportuj plik JSON.
3. Zweryfikuj, czy App Pool account jest Managed Account w farmie.
4. Uruchom `DryRun`.
5. Przejrzyj raport HTML i ostrzeżenia.
6. Dopiero potem uruchom `Execute`.

## Polecenia startowe

```powershell
# Tryb podglądu
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode DryRun

# Tryb wykonania
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode Execute

# Własny katalog logów
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode DryRun -LogDirectory "C:\SP_Logs"

# Bez interaktywnego potwierdzenia
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode Execute -Force

# Z pominięciem walidacji
.\Start-SPProvisioning.ps1 -ConfigFile ".\config.json" -Mode Execute -SkipValidation
```

## Checklista przed `Execute`

- JSON został wygenerowany z aktualnej wersji formularza.
- `DryRun` zakończył się bez błędów krytycznych.
- Port Web Application nie jest zajęty.
- Managed Account dla nowego App Pool istnieje w farmie.
- Szablony witryn są dostępne dla wskazanego języka.
- SQL Server jest osiągalny z serwera SharePoint.
- Uprawnienia właścicieli i grup są poprawne domenowo.
- `managedPath` oraz `managedPathType` odpowiadają planowanemu URL.

## Checklista po wykonaniu

- Raport HTML ma status bez błędów krytycznych.
- Log tekstowy nie zawiera nieoczekiwanych `ERROR`.
- Site Collections są dostępne pod oczekiwanymi adresami.
- Podwitryny zostały utworzone na poprawnych ścieżkach.
- Listy i biblioteki są widoczne we właściwych witrynach.
- Uprawnienia zostały przypisane zgodnie z konfiguracją.

## Gdzie szukać wyników

| Artefakt | Domyślna lokalizacja |
| --- | --- |
| Log | `Logs\SP_Provisioning_*.log` |
| Raport HTML | `Logs\SP_Provisioning_Report_*.html` |
| Dokumentacja DOCX | `SP-Framework-Dokumentacja.docx` |

## Diagnostyka środowiska

### SharePoint PowerShell

```powershell
Get-PSSnapin -Registered | Where-Object { $_.Name -eq "Microsoft.SharePoint.PowerShell" }
Get-SPFarm
```

### Managed Accounts

```powershell
Get-SPManagedAccount | Select-Object UserName
```

### Web Applications i porty

```powershell
Get-SPWebApplication | Select-Object DisplayName, Url
```

### Managed Paths

```powershell
Get-SPManagedPath -WebApplication "http://intranet.firma.local"
```

### Site Collections

```powershell
Get-SPSite -Limit All | Select-Object Url, ContentDatabase
```

### Content Databases

```powershell
Get-SPContentDatabase -WebApplication "http://intranet.firma.local" |
    Select-Object Name, DatabaseServer, CurrentSiteCount, WarningSiteCount
```

## Typowe problemy

| Objaw | Prawdopodobna przyczyna | Zalecane działanie |
| --- | --- | --- |
| Błąd ładowania snapina SharePoint | Skrypt uruchomiony poza serwerem SharePoint albo bez poprawnego środowiska | Użyj SharePoint Management Shell albo sprawdź rejestrację snapina |
| Błąd Managed Account | Konto App Pool nie jest zarejestrowane w farmie | Zarejestruj konto w SharePoint przed uruchomieniem `Execute` |
| Konflikt portu | Port jest już używany przez inną Web Application | Zmień port w konfiguracji albo zweryfikuj istniejące WA |
| Site Collection nie powstaje | Błędny `managedPath` albo kolizja URL | Zweryfikuj `managedPath`, `managedPathType` i `url` |
| Podwitryna trafia pod złą ścieżkę | `subsite.url` nie zawiera pełnej ścieżki od korzenia Site Collection | Użyj pełnych URL, np. `/it/helpdesk` |
| Brak dostępu do witryny po utworzeniu | `inheritPermissions = false` bez poprawnych wpisów | Dodaj co najmniej jedną poprawną rolę dla użytkownika lub grupy |
| Błąd SQL | Niedostępny serwer, błędna instancja albo brak uprawnień | Zweryfikuj połączenie z SQL i uprawnienia serwisowe |

## Notatki do `DryRun`

`DryRun` jest najlepszym trybem do bezpiecznej weryfikacji logiki, ale trzeba pamiętać o jego granicach:

- nie tworzy żadnych obiektów w farmie
- nie wykonuje pełnego testu połączenia z SQL
- nie sprawdza wszystkich kolizji istniejących obiektów SharePoint
- generuje raport HTML, który nadaje się do review przed wykonaniem zmian

## Notatki do rollbacku

Rollback działa automatycznie tylko dla części scenariuszy i tylko w kontekście sesji, która właśnie się wykonuje.

- Kolejność wycofania jest odwrotna do kolejności tworzenia obiektów.
- Osierocona dedykowana Content Database po nieudanym tworzeniu Site Collection jest usuwana od razu.
- Usunięcie Web Application nie czyści automatycznie bazy SQL.
- Przy awarii po częściowo wykonanych operacjach zawsze warto przejrzeć raport HTML i log przed ręcznym cleanupem.

## Generowanie dokumentacji DOCX

```powershell
.\Run-GenerateDocumentation.ps1
```

Uwagi:

- Wrapper dodaje UTF-8 BOM do tymczasowej kopii skryptu.
- Docelowy plik to `SP-Framework-Dokumentacja.docx`.
- Generator nie wymaga Microsoft Word.

## Minimalny runbook administratora

1. Otwórz formularz i zbuduj JSON.
2. Uruchom `DryRun`.
3. Przejrzyj `WARNING` i `ERROR`.
4. Potwierdź dostępność Managed Account, SQL i szablonów.
5. Uruchom `Execute`.
6. Zarchiwizuj raport HTML i log tekstowy jako ślad wdrożenia.
