# Hone – Sprachgesteuerte Texteingabe und Textbearbeitung

## Zweck und Abgrenzung

Hone verbindet bestehende Programme für Spracherkennung, Textverarbeitung und Texteingabe unter KDE Wayland. Es ermöglicht schnelle Diktate ohne LLM sowie gezielt aufrufbare Nachbearbeitung und sprachgesteuerte Bearbeitung markierter Texte.

Hone ist eine schlanke, eigenständig aufrufbare CLI. Es bringt keine eigene Spracherkennung, Audioaufnahme, Tastaturüberwachung oder Clipboard-Verwaltung mit und benötigt keinen eigenen Daemon. Globale Tastenkürzel werden über die Desktopumgebung eingerichtet. Die verwendeten Programme sind über konfigurierbare Befehle austauschbar; Voxtype, Ollama, wl-clipboard und eitype bilden die vorgesehene Integration.

## Bedienung

Vier Funktionen stehen unabhängig voneinander zur Verfügung und können jeweils einem eigenen globalen Tastenkürzel zugeordnet werden.

| Befehl | Funktion | Eingabe | Verarbeitung |
| --- | --- | --- | --- |
| `hone dictate --raw` | Normales Diktat | Sprache | Transkription ohne LLM |
| `hone dictate` | Diktat mit Cleanup | Sprache | Transkription und Cleanup |
| `hone clip` | Clipboard-Cleanup | Text aus der Zwischenablage | Cleanup |
| `hone edit` | Kommandomodus | Markierter Text und gesprochene Anweisung | Bearbeitung des Textes gemäß der Anweisung |

Alle vier Funktionen werden von Hone abgewickelt. Bei den sprachbasierten Funktionen startet der erste Aufruf die Aufnahme; der zweite Aufruf desselben Befehls beendet sie und übernimmt die fertige Transkription. Clipboard-Cleanup benötigt keine Aufnahme und wird mit einem Aufruf ausgeführt.

Eine leere Transkription führt weder zu einer Ausgabe noch zu einem Aufruf der Textverarbeitung. Während der LLM-Verarbeitung gibt Hone eine sichtbare Rückmeldung.

### Textausgabe

Alle vier Funktionen fügen das Ergebnis über die Zwischenablage in die aktive Anwendung ein. Es findet keine zeichenweise Simulation der Texteingabe statt. Das Ergebnis wird nicht lediglich auf stdout oder in der Zwischenablage ausgegeben.

Ist an der Eingabeposition eine Textauswahl vorhanden, wird diese durch das Ergebnis ersetzt. Mehrzeilige Texte werden als Text eingefügt, nicht als Folge einzelner Enter-Tastendrücke. Die ursprüngliche Zwischenablage wird anschließend gemäß den Regeln im Abschnitt „Zwischenablage“ wiederhergestellt. Die Zielanwendung erhält beim Einfügen das Ergebnis und nicht den vorherigen Clipboard-Inhalt.

Die Ausgabe erfolgt an der zum Zeitpunkt des Einfügens aktiven Eingabeposition. Hone verfolgt weder Fenster noch Textauswahlen. Geht die ursprüngliche Auswahl während der Verarbeitung verloren, wird das Ergebnis an der dann aktiven Eingabeposition eingefügt.

Hone verwendet einen global konfigurierbaren Einfügebefehl. Anwendungen mit davon abweichenden Einfügekürzeln, insbesondere Terminals, können eine andere Einstellung erfordern. Eine automatische Unterscheidung nach Anwendung findet nicht statt.

## Normales Diktat

Hone übernimmt die abgeschlossene Transkription und fügt sie ohne LLM-gestützte Nachbearbeitung ein. Das Diktierwerkzeug gibt den Text nicht selbst in die aktive Anwendung aus. Die Funktion ist unabhängig von der Verfügbarkeit des Textverarbeitungs-Backends.

## Diktat mit Cleanup

Hone übernimmt die abgeschlossene Transkription, ohne dass das Diktierwerkzeug sie selbst in die aktive Anwendung einfügt.

Der erkannte Text wird mit dem Cleanup-Prompt verarbeitet. Bei erfolgreicher Verarbeitung wird nur der nachbearbeitete Text eingefügt, nicht zusätzlich die ursprüngliche Transkription.

Schlägt die Textverarbeitung fehl, beispielsweise weil das Backend nicht erreichbar ist oder nicht rechtzeitig antwortet, wird die unbearbeitete Transkription eingefügt und der Fehler angezeigt.

## Clipboard-Cleanup

Hone liest den aktuellen Textinhalt der normalen Zwischenablage und verwendet denselben Cleanup-Prompt wie beim Diktat mit Cleanup.

Das Ergebnis wird in die aktive Anwendung eingefügt. Der ursprüngliche Clipboard-Inhalt bleibt nach der Wiederherstellung gemäß den Regeln im Abschnitt „Zwischenablage“ erhalten. Der Modus kann auch für Texte verwendet werden, die nicht durch Diktat entstanden sind.

Enthält die Zwischenablage keinen Text, wird der Vorgang mit einem Hinweis abgebrochen. Schlägt die Textverarbeitung fehl, wird der Fehler angezeigt und nichts eingefügt.

## Cleanup-Verhalten

Cleanup korrigiert die Eingabe, ohne sie inhaltlich umzuschreiben:

- Rechtschreibung, Grammatik, Zeichensetzung und offensichtliche Spracherkennungsfehler werden korrigiert; Füllwörter und erkennbare Fehlstarts werden entfernt.
- Die ursprüngliche Sprache, Bedeutung und möglichst auch die Formulierung bleiben erhalten. Bereits korrekter Text bleibt unverändert.
- Technische Begriffe, API-Namen, Klassen, Methoden, Variablen und Dateipfade werden als solche erkannt. Gesprochene Bezeichner werden bei eindeutigem Kontext in übliche Entwicklernotation übertragen, einschließlich PascalCase, camelCase, snake_case, kebab-case und UPPER_SNAKE_CASE.
- Qualifizierte Bezeichner behalten ihre Trennzeichen. Bei mehrdeutigen Bezeichnern werden keine Namen erfunden und keine unbegründeten Änderungen vorgenommen.
- Die Ausgabe enthält ausschließlich den bereinigten Text. Es werden keine Erklärungen, einleitenden Sätze, umschließenden Anführungszeichen oder Markdown-Codeblöcke hinzugefügt, die nicht zum eigentlichen Text gehören. Bereits vorhandene, zum Text gehörende Formatierung bleibt erhalten.

Beispiele:

| Eingabe | Erwartetes Ergebnis |
| --- | --- |
| `ähm das müssen wir morgen nochmal prüfen` | `Das müssen wir morgen nochmal prüfen.` |
| `Router Service Punkt on routes recognized` (als qualifizierter Klassen- und Methodenname diktiert) | `RouterService.onRoutesRecognized` |
| `Die Methode liefert das Ergebnis als JSON.` | `Die Methode liefert das Ergebnis als JSON.` |

## Kommandomodus

Der Benutzer markiert den zu bearbeitenden Text in der aktiven Anwendung und startet den Kommandomodus. Hone übernimmt die aktuelle Textauswahl zu Beginn der Aufnahme. Ein zuvor kopierter Text darf nicht fälschlich als aktuelle Auswahl behandelt werden. Ist kein Text ausgewählt, wird der Vorgang mit einem Hinweis abgebrochen, ohne die Textverarbeitung aufzurufen.

Der Benutzer spricht die gewünschte Bearbeitung. Nach Abschluss der Aufnahme übernimmt Hone die Transkription als Anweisung. Der ausgewählte Text und die gesprochene Anweisung werden getrennt an die Textverarbeitung übergeben. Der Kommandomodus verwendet einen eigenen Prompt, der die gewünschte Textbearbeitung ausführt, anstatt ausschließlich Cleanup anzuwenden.

Die Textverarbeitung liefert ausschließlich den gewünschten bearbeiteten Text zurück, ohne zusätzliche Erklärungen oder unerwünschte Formatierungen. Ausdrücklich gewünschte Formatierungen, beispielsweise ein Markdown-Codeblock, gehören zum Ergebnis und werden beibehalten.

Hone fügt das Ergebnis ein und ersetzt damit die noch bestehende Textauswahl. Die ursprüngliche Zwischenablage wird sowohl nach der Übernahme der Auswahl als auch nach dem Einfügen gemäß den Regeln im Abschnitt „Zwischenablage“ wiederhergestellt. Schlägt die Textverarbeitung fehl, wird der Fehler angezeigt und nichts eingefügt; der ausgewählte Text bleibt unverändert.

Beispiele:

| Markierter Text | Gesprochene Anweisung | Erwartetes Ergebnis |
| --- | --- | --- |
| `Das Feature funktioniert nicht richtig und wir müssen uns das nochmal anschauen.` | „Übersetze das ins Englische und formuliere es als GitHub-Issue-Titel.“ | `Investigate and fix incorrect feature behavior` |
| `Die Methode funktioniert nicht.` | „Übersetze das ins Englische.“ | `The method does not work.` |

Die Beispiele für Cleanup und Kommandomodus veranschaulichen das erwartete Verhalten; sie schreiben keine exakten Modellantworten für jede mögliche Eingabe vor.

Die gesprochene Anweisung bezieht sich auf die Bearbeitung von Text. Hone führt keine vom Modell erzeugten Shell-Befehle aus und nimmt keine vom Modell veranlassten Aktionen außerhalb der Textbearbeitung vor.

## Zwischenablage

Die Zwischenablage wird nur vorübergehend zur Übernahme einer Textauswahl und zum Einfügen des Ergebnisses verwendet. Der vorherige Inhalt wird nach diesen Vorgängen wiederhergestellt, auch wenn die Verarbeitung abgebrochen wird oder fehlschlägt. Die Wiederherstellung nach dem Einfügen darf nicht dazu führen, dass die Zielanwendung statt des Ergebnisses den vorherigen Inhalt einfügt.

Insbesondere darf bei der Wiederherstellung von Text nicht HTML-Quelltext anstelle des ursprünglichen Klartexts in der Zwischenablage landen. Diese Anforderung adressiert den bereits beobachteten Fehler bei Voxtype.

Die vorgesehene wl-clipboard-Integration kann beim Wiederherstellen nur eine Nutzdaten-Repräsentation pro Aufruf anbieten. Eine vollständige Wiederherstellung aller gleichzeitig angebotenen MIME-Repräsentationen ist daher nicht zugesichert; bei Text wird eine Klartext-Repräsentation bevorzugt.

Die von Hone vorübergehend zum Einfügen in die Zwischenablage geschriebenen Ergebnisse sollen nicht im KDE-Clipboard-Verlauf (Klipper) erscheinen. Hone kennzeichnet diese Inhalte als sensibel. Für Text, den die aktive Anwendung beim Kopieren der Auswahl im Kommandomodus selbst in die Zwischenablage schreibt, wird dies nicht zugesichert. Beim Wiederherstellen ursprünglich sensibler Clipboard-Inhalte bleibt deren sensible Kennzeichnung erhalten. Für andere Clipboard-Manager wird das Verhalten des Verlaufs nicht zugesichert.

## Integration und Konfiguration

### Spracherkennung

Voxtype ist für Aufnahme und Spracherkennung zuständig. Seine integrierte LLM-Nachbearbeitung ist deaktiviert.

Hone übernimmt die Transkription für alle sprachbasierten Funktionen über die öffentliche Voxtype-CLI. Die Aufnahme wird mit einer nur für diese Aufnahme geltenden Dateiausgabe gestartet (`record start --file=<Pfad>`) und über `record stop --wait` beendet. Der fertige Text wird über stdout übernommen. Der normale Voxtype-Ausgabemodus bleibt unverändert; ein eigenes Voxtype-Profil ist nicht erforderlich.

Die Befehle zum Starten und Beenden der Aufnahme sind konfigurierbar. Ein alternatives Diktierwerkzeug muss eine entsprechende Aufnahmesteuerung und die Übernahme der abgeschlossenen Transkription ohne eigene Textausgabe in die aktive Anwendung ermöglichen.

### Textverarbeitung

Ollama wird über eine OpenAI-kompatible HTTP-Schnittstelle angesprochen. Server-URL und Modell sind zentral konfigurierbar. Eine lokale Ollama-CLI ist nicht erforderlich.

Der gemeinsame Cleanup-Prompt und der Prompt für den Kommandomodus sind separat konfigurierbar. Die Textverarbeitung ist nicht auf einen bestimmten Anbieter festgelegt; alternativ kann ein konfigurierbarer externer Befehl die Verarbeitung übernehmen. Dieser erhält den zu verarbeitenden Text und im Kommandomodus zusätzlich die davon getrennte Anweisung. Das Ergebnis liefert er als Text über stdout.

### Zwischenablage und Tastensimulation

Die Befehle zum Lesen und Schreiben der Zwischenablage sowie zum Auslösen von Kopieren und Einfügen sind konfigurierbar. Die vorgesehene Integration verwendet `wl-paste`, `wl-copy` und `eitype`. eitype dient ausschließlich der Simulation von Tastenkombinationen. Für Hone sind kein Zugriff auf `/dev/uinput`, keine Mitgliedschaft in der `input`-Gruppe und kein privilegierter Eingabedienst erforderlich.

Zur Übernahme der Textauswahl wird der in Murmure verwendete Ansatz genutzt: Zwischenablage sichern, leeren, Kopierbefehl auslösen, den neu kopierten Text übernehmen und die ursprüngliche Zwischenablage wiederherstellen.

Zur Ausgabe wird das Ergebnis vorübergehend in die Zwischenablage geschrieben, der Einfügebefehl ausgelöst und anschließend der ursprüngliche Inhalt wiederhergestellt. Die vorgesehene Kennzeichnung temporärer Ergebnisse als sensibel erfolgt über `wl-copy --sensitive` (KDE-MIME-Hinweis `x-kde-passwordManagerHint`).

## Projekt

Hone wird in einem eigenständigen GitHub-Repository geführt. Diese Spec ist der fortlaufend gepflegte Produktvertrag. Die Benutzerdokumentation beschreibt Installation, Konfiguration und Verwendung.
