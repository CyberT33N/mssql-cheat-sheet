
### Docker (falls MSSQL in Container läuft)

Für den lokalen Dev-Container `mssql-dev` mit `sa` / `Test1234!` und der Datenbank `Z1`.

Wenn die SQL-Datei bestehende Objekte wie Tabellen erneut anlegt, muss die Datenbank vorher gelöscht und neu erstellt werden. Sonst kommen Fehler wie `There is already an object named ... in the database`.

```powershell
# Datei in den Container kopieren
docker cp "C:\git\test\test-mono\apps\test\data\dumps\pvs\z1\base\sql\without-timer\all-in-one\z1_base_2026-03-06_15_50_02.sql" mssql-dev:/tmp/z1_base_2026-03-06_15_50_02.sql

# Bestehende Verbindungen beenden, Datenbank Z1 löschen und neu anlegen
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d master -No -C -Q "IF DB_ID(N'Z1') IS NOT NULL BEGIN ALTER DATABASE [Z1] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [Z1]; END; CREATE DATABASE [Z1];"

# SQL-Datei in die neu erstellte Datenbank Z1 importieren
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d Z1 -No -C -t 0 `
    -i /tmp/z1_base_2026-03-06_15_50_02.sql
```

Falls du es als Einzeiler ausführen willst:

```powershell
docker cp "C:\git\test\test-mono\apps\test\data\dumps\pvs\z1\base\sql\without-timer\all-in-one\z1_base_2026-03-06_15_50_02.sql" mssql-dev:/tmp/z1_base_2026-03-06_15_50_02.sql; docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Test1234!" -d master -No -C -Q "IF DB_ID(N'Z1') IS NOT NULL BEGIN ALTER DATABASE [Z1] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [Z1]; END; CREATE DATABASE [Z1];"; docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Test1234!" -d Z1 -No -C -t 0 -i /tmp/z1_base_2026-03-06_15_50_02.sql
```









<br><br>


## Troubleshooting $(...)-Sequenzen
- Wenn man den Export selbst erstellt, muss man darauf achten, dass die Inhalte korrekt escaped werden, weil ein Dollarzeichen mit einer offenen Klammer danach bei sqlcmd zu Problemen führen kann. Dadurch können beim Import Syntaxfehler auftreten.

Das heißt, man muss erstens sicherstellen, dass der SQL-Dump entsprechend korrekt ist.

Falls man aber einen Invaliden hat, kann man das umgehen, indem man mit dem X-Flag arbeitet.





Ja, jetzt ist die Ursache fachlich klar: Das Problem liegt **im Exportformat für `sqlcmd`**, nicht in einem normalen SQL-`INSERT`-Escaping.

Die entscheidende Beobachtung ist, dass eure exportierten Daten **literale `$(...)`-Sequenzen** enthalten, zum Beispiel:
- `N'$('`
- `N'$(Vorbreitung auf ZE) '`
- `N' N nach $( EX , 16,17 Fllg je 100,- UPT  '`

`sqlcmd` interpretiert `$(...)` aber als **Scripting-Variable**, also **vor** der eigentlichen SQL-Ausführung. Deshalb kommt der Fehler `Syntax error near command '('`. Das ist kein normaler T-SQL-Stringfehler, sondern ein `sqlcmd`-Preprocessor-Thema.

Der Export splittet solche Werte jetzt in mehrere Unicode-Stringteile, damit `sqlcmd` kein rohes `$(...)` mehr sieht, der Datenwert aber identisch bleibt.

Zusätzlich bleibt der **Importer-Fix mit `-x`** richtig und wichtig:
- Der Importer deaktiviert `sqlcmd`-Variablenersetzung jetzt ebenfalls.
- Das ist die richtige Defense-in-depth.
- Dein direkter Docker-Test hat noch **ohne `-x`** gearbeitet, deshalb ist er weiter auf den Fehler gelaufen.

Für den **aktuellen schon erzeugten Dump** gilt:
- Der ist bereits mit dem alten fehlerhaften Export erzeugt worden.
- Wenn du ihn direkt mit `sqlcmd` testen willst, nimm `-x` dazu.
- Für einen wirklich sauberen Zustand solltest du den Dump **neu exportieren** mit dem jetzt gefixten Exporter.

Für deinen Docker-Test heißt das konkret: statt

```powershell
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d Z1 -No -C -t 0 `
    -i /tmp/z1_base_2026-03-06_15_50_02.sql
```

brauchst du mit dem alten Dump mindestens:

```powershell
docker exec -it mssql-dev /opt/mssql-tools18/bin/sqlcmd `
    -S localhost -U sa -P "Test1234!" `
    -d Z1 -No -C -t 0 -x `
    -i /tmp/z1_base_2026-03-06_15_50_02.sql
```

Kurz gesagt:
- **Ja**, dein Gedanke war valide.
- Das Problem ist **nicht** primär der Restore.
- Das Problem ist, dass euer **Export bisher nicht `sqlcmd`-sicher** war.
- Der Importer wurde bereits gehärtet.
- Der Exporter ist jetzt ebenfalls gehärtet.
- Für den aktuellen Dump: `-x` verwenden oder Dump neu erzeugen.

Der nächste sinnvolle Schritt ist jetzt:
1. den Base-Dump **neu exportieren**
2. danach den Import nochmal laufen lassen
3. alternativ den bestehenden Dump einmal testweise mit `-x` direkt in Docker ausführen, um zu bestätigen, dass genau diese Ursache den Fehler auslöst