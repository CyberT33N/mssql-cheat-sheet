## SQL Server Management Studio (SSMS) — SQL-Dateien importieren

Für große SQL-Dateien (`.sql`) gibt es verschiedene Import-Methoden. SSMS selbst hat bei sehr großen Dateien (> 100 MB) oft Probleme mit dem Editor.


**WICHTIG**
- Also, weder mit Option 1 noch mit Option 2 unten — also weder über eine normale Query noch, wenn man die Query im sqlcmd-Modus ausführt — ist es möglich, sehr große Dumps zu importieren.

---

### Option #1 - Direkter Query (kleine Dateien bis ~50 MB)

Du musst sie **nicht** per Copy-and-Paste ins Clipboard nehmen.

In SSMS wäre der normale Weg:

1. Mit dem Server verbinden.
2. Im `Object Explorer` die bestehende Zieldatenbank auswählen.
3. Rechtsklick auf die Datenbank und `New Query`.
4. Im Query-Fenster oben auf `File > Open > File...` gehen.
5. Die `.sql`-Datei auswählen.
6. Oben prüfen, dass als Datenbank wirklich deine Ziel-Datenbank gesetzt ist.
7. Mit `Execute` oder `F5` ausführen.

Der wichtige Punkt ist: **Die Datei wird direkt von der Festplatte geöffnet**, nicht über die Zwischenablage.


---

### Option #2 - SQL CMD Mode


Für **1,8 GB** ist aber die eigentliche Einschränkung: **SSMS ist dafür in der Regel nicht geeignet**. Das Öffnen kann extrem langsam sein, hängen oder abstürzen. Technisch ist der Ablauf also `New Query -> Open File -> Execute`, aber praktisch ist diese Dateigröße für den SSMS-Editor meist zu groß.

Wenn du es unbedingt in SSMS-naher Form machen willst, gibt es noch diese Variante mit **SQLCMD Mode** in SSMS:

1. Datei
2. Neu > Abfrage mit aktueller Verbindung
3. Abfrage > SQL CMD Modus

```sql
USE [DeineDatenbank];
GO
:r C:\Pfad\zu\deiner\datei.sql
GO
```

Damit referenzierst du die Datei vom Dateisystem, statt sie manuell reinzukopieren. Aber auch hier gilt: Bei **1,8 GB** kann SSMS trotzdem problematisch sein.

Für diese Größe ist der robuste Weg normalerweise **nicht der SSMS-Editor**, sondern `sqlcmd` direkt gegen die bestehende Datenbank.

Wenn du möchtest, formuliere ich dir im nächsten Schritt die **exakten SSMS-Schritte nur für deine bestehende Datenbank**, ganz ohne `sqlcmd`, und sage dir auch direkt, an welcher Stelle du sehr wahrscheinlich scheitern wirst bei 1,8 GB.