# Wissen und Regeln für Claude – FBG-Tool

Diese Datei liest Claude zu Beginn jeder Sitzung automatisch. Sie ersetzt den langen Entwicklungs-Chat.

## Wer und wofür

- Nutzer: Kai („Meister“), IT-Admin und Lehrer am Friedrich-Bährens-Gymnasium (FBG) Schwerte, NRW. Kein Programmierer.
- Ansprache: duzen, Deutsch, einfache Sprache, kurz. Oberflächen und Anleitungen bevorzugt, nummerierte Schritte.
- **PowerShell-Befehle immer vollständig und kopierbar ausgeben**, nie „zweiter Lauf ohne Parameter …“.
- Sagt er „ändere noch nichts“ oder „leg mir erst die Vorschläge vor“: erst eine Liste vorlegen, dann auf sein Okay warten.
- Entscheidungen über SchILD-Rückimporte trifft er nicht allein: „Alles muss ich erst abklären.“
- GitHub: Kai ist Anfänger. Claude arbeitet auf dem eigenen Branch und pusht nie direkt nach `main`. Fertige Pakete gehen gesammelt als **ein Pull Request** raus; Claude fasst den Inhalt kurz zusammen; erst wenn Kai „merge“ schreibt, merged Claude den Pull Request selbst. Nie ohne dieses Wort. Schritte dafür immer kurz erklären.

## Datenschutz (wichtigste Regel)

- Das Tool verarbeitet echte Schüler- und Lehrerdaten. Es läuft 100 % offline im Browser. **Echte Daten kommen nie ins Repo und nie in den Chat.** Nur erfundene Testdaten.
- Keine echten Namen in Beispieldaten. Die Testlehrkräfte heißen Ahorn, Birke, Eiche, Linde, Ulme, Weide.
- Alle erzeugten Dateien gehen in den Download-Ordner. Im gemeinsamen Ordner liegt nur `FBG_Schuljahr_Stand.json`. Dateien mit Klartext-Passwörtern nach dem Verteilen löschen.

## Technik

- Eine einzige Datei `FBG_Schuljahr.html`, Vanilla JS, kein Build, keine externen Bibliotheken. Chrome/Edge (File System Access API).
- Stand: Objekt `STAND`, gespeichert in `FBG_Schuljahr_Stand.json` + localStorage. Neue Schlüssel an **allen drei** Stellen mit Vorgabewert anlegen (z. B. `praktika:{}`, `vorlagen:[]`, `abgleich:{}`).
- Modi: `state.mode` = `schueler` | `lehrer` | `praktikum`.
- Abläufe: `VORGAENGE` (art `jahr`/`laufend`) mit `SCHRITTE` {id, titel, reiter, ort, dauer, zweck, anleitung[], warum}. Daraus werden Reiter 0 „Abläufe und Anleitungen“, das Handbuch und die Suche erzeugt.
- Suche: `suchIndex()`, `suchLauf()`. Briefvorlagen: `VORLAGEN_STD`, Platzhalter über `briefFelder()`; Datum immer TT.MM.JJJJ (`datumDe`).
- Download: nur eine Funktion `download()` → `speichereDatei(..., true)` = Downloads.
- Nach jeder Änderung: Version im `<title>` und `<h1>` erhöhen (aktuell v4.66), Selbsttest (Einstellungen → Selbsttest) muss grün bleiben (87/87). Testen z. B. mit Node + jsdom oder Playwright (Chromium unter `/opt/pw-browsers/chromium`): `selbsttest(false)`.
- PowerShell-Variablen sind nicht case-sensitiv: `$Soll` und `$soll` sind dieselbe Variable. Selbsttest prüft das. Erzeugte Skripte lassen sich lokal mit pwsh und Stub-Funktionen für die Exchange-Cmdlets testen. Fehlt pwsh in der Cloud-Sitzung, nachinstallieren: `mkdir -p /opt/pwsh && curl -sSL https://github.com/PowerShell/PowerShell/releases/download/v7.4.6/powershell-7.4.6-linux-x64.tar.gz | tar xz -C /opt/pwsh && chmod +x /opt/pwsh/pwsh`. Erzeugte Skripte als eigenes Skript starten (`./x.ps1`), nicht dot-sourcen, sonst überschreiben ihre Variablen die Stub-Variablen.
- Achtung bei Ersetzungen: `</style>` kommt mehrfach vor (auch in erzeugten Handbuch-/Brief-Strings).

## Fachwissen Systeme

- **SchILD-NRW 2.0.33.8**, MariaDB `schild_nrw` (ODBC „FBG“) auf eigener VM in XCP-ng. Vor Importen: Snapshot genügt. SchILD 3 läuft parallel an. „EF“ in SchILD wird nur für Jamf zu „EP“ (Einstellung `jgUm`).
- **MS365 / Graph**: leere Strings (z. B. Department) werden abgelehnt → Parameter per Splat, leere weglassen, `New-MgUser` in try/catch mit Fehlerzähler. `ForceChangePasswordNextSignIn=$true`.
- **Praktikanten** (Praxissemester, ca. 5 pro Halbjahr): MS365-Konto, Kollegiumsverteiler, Position aus `prakPosition`, Lizenz `skuPrak`, befristet. Kennung `PRAKT-<Kohorte>-NN` in extensionAttribute1, Enddatum ISO in extensionAttribute2. Kein IServ, kein Jamf. WebUntis per SSO, eigene Benutzergruppe (`untisGruppe`).
- **Jamf School**: Benutzerimport ersetzt Gruppen; Schlüssel = Username. Gruppen ≠ Klassen (Migration manuell, erneute Migration schadet nicht). TeacherGroups-CSV ist gruppenorientiert: `TeacherGroup;UserNames` (eine Gruppe pro Zeile, Namen mit Komma). Lehrernamen aus dem Jamf-Lehrerexport nehmen, nicht nach Regel bilden. Sek I = Jahrgang 05–10.
- **Ibiza-EP-Export**: 7er-Blöcke ab Spalte 7 (Fach, Fach kurz, Kurszeichen, Kürzel, Kursnummer, Block, Fach/Art/Block). Gruppenname `EP_<FachKurz>_<Rest>`, z. B. `EP_D_GK8`. Ignorierte Kurse: Einstellung `kursIgnore`.
- **WebUntis**: Import unter Administration → Benutzer → Benutzerverwaltung → Import. Schlüssel = Benutzername (Fremdbenutzername ist KEIN Schlüssel). Benutzernamen ca. vorname.nachname. Schüler und Praktikanten melden sich per MS365-SSO an (Feld „Office 365 Identität“). Schüler-Stammdatenbericht liefert `externKey` = SchILD-ID.
- **Klassenabgleich** (Reiter 3, vereinbart 04.10.2026): prüft SchILD-Klassen gegen die Klassenverteiler 05a–10e **und** EP/Q1/Q2. Kern ist `abgKern()` (ohne Oberfläche, im Selbsttest geprüft).
  - In Klassenverteiler gehören nur Kinder der Klasse und Lehrkräfte (Klassenleitungen). Lehrkraft = steht im **Kollegiumsverteiler** (Einstellung `verteiler`; Fenster 2 liest ihn mit, Zeilen mit Verteiler `#Kollegium`, Untergruppen eine Ebene tief) oder Position enthält „Lehr“/„Referend“ oder steht in der Lehrerliste der Karte Klassenleitungen. Lehrkräfte werden nie gemeldet. Die Position allein reicht nicht: Ältere Lehrerkonten haben dort nichts stehen (gemeldet von Kai am 04.10.2026, behoben in v4.65).
  - Abgänge werden aus ihren Klassenverteilern entfernt (vorausgewählt, Fehler sind leicht zu beheben). Das Konto bleibt immer unberührt.
  - Kinder in falschen Verteilern (Wechsel oder zusätzlich drin) sind vorausgewählt. Unbekannte Mitglieder werden nur gezeigt, nie vorausgewählt.
  - Alles Angehakte geht in **ein** Korrekturskript `6_Klassenwechsel.ps1`. Es ändert nur Verteiler.
  - Fenster 1 liest zusätzlich `Mail` (Verteiler liefern die Mailadresse, nicht den UPN). Fenster 2 schreibt auch leere Verteiler (Zeile mit leerer Adresse) und die Spalte `Name`.
  - Die Startseite erinnert 4 Wochen nach dem Haken bei „Verteiler versetzen“, Datum der letzten Prüfung in `STAND.abgleich`.
  - Die Karte „So gehst du vor“ im Reiter Abgleich zeigt die `anleitung[]` des Schritts `abgleich` (`abgAnleitungZeichnen()`). Anleitung nur dort in `SCHRITTE` pflegen, nie doppelt im HTML.
- **WLAN**: persönlich `FBG` (RADIUS, IServ-Zugang), Geräte `GeräteFBG` (festes Passwort, Einstellung `wlanPasswort`).

## Offene Punkte (Stand 04.10.2026)

- Anleitungen (`anleitung[]`) für die übrigen Vorgänge schreiben: Schuljahreswechsel, Oberstufenkurse, neue Schüler, neue Lehrkraft. Entwurf vorlegen, Kai korrigiert.
- Anleitung zum Schritt „Änderungen abgleichen“ (14 Schritte, seit v4.66 auch auf der Karte „So gehst du vor“): Kai prüft sie beim ersten echten Lauf und meldet Korrekturen.
- Abgleich auf Jamf/IServ erweitern + Korrekturdateien (Klassenverteiler sind seit v4.64 erledigt); passwortfreie IServ-Standdatei; Merge-Skript in den Werkzeugkasten.
- Werkzeugkasten: alle Karten beim Start zugeklappt.
- WebUntis-Karte: „Vorname gekürzt“ vorauswählen (Sek I wird ab nächstem Jahrgang direkt in WebUntis importiert).
- Warnung, wenn Kursspalten leer sind; Hinweis Zeugnisdaten Schuljahr/Halbjahr; Exportvorlage sichern.
- Handbuch-Abschnitt „Nachträgliche Namensänderung“.
- Voreinstellung `kursIgnore` → `AG,SpKl`.
- Lehrkräfte-Abgleich in Reiter 3.
- Klassenleitungen aus SchILD (erst mit SchILD 3).
- Karte L: Dienstadresse aus einer E-Mail-Spalte der Lehrerliste lesen, sobald die Dienstadressen in SchILD gepflegt sind (bis dahin: Adresse je Kürzel von Hand ändern).
- Später: Jahresübersicht; Grundordnung der Reiter überdenken, wenn alle Vorgänge aufgenommen sind.
