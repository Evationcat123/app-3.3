# Browser-Restart-Taste & Backspace-Long-Press Patch

## Neue Taste „Browser neu starten“ (⟳)
- Neuer Button in der Browser-Toolbar neben der bestehenden Reload-Taste (↻), optisch im gleichen Stil (`smallButton`, folgt dem Theme über `styleSmallButton`).
- Öffnet **niemals** eine externe Browser-App und öffnet keine zusätzliche Activity — bleibt vollständig innerhalb des `InputMethodService`.
- `restartBrowser()`:
  - entfernt die aktuelle WebView sauber aus dem Layout (an der ursprünglichen Position gemerkt),
  - lässt `createWebView()` die alte, bereits abgehängte Instanz zerstören (kein Leak) und eine neue erzeugen,
  - fügt die neue WebView wieder an derselben Stelle ein und lädt die zuletzt angezeigte Seite (`lastWebUrl`) neu,
  - aktualisiert das Adressfeld.
- Da `createWebView()` selbst vor jedem Neuaufbau eine evtl. vorhandene WebView zerstört, entstehen auch bei mehrfachem, schnellem Drücken keine doppelten WebView-Instanzen/Tabs und keine Memory-Leaks.
- Ist beim Drücken aus irgendeinem Grund keine WebView vorhanden, erzeugt derselbe Codepfad einfach eine neue — die Taste „öffnet“ den Browser in diesem Fall.

## Backspace: gedrückt halten zum wiederholten Löschen
- Die Backspace-Taste nutzt jetzt einen eigenen `OnTouchListener` (`addBackspaceKey`) statt eines einfachen Klick-Listeners:
  - `ACTION_DOWN`: löscht sofort ein Zeichen/eine Markierung (normales kurzes Tippen) und startet einen verzögerten Wiederhol-Timer.
  - Nach ca. 400 ms beginnt automatisches, kontinuierliches Löschen; das Intervall verkürzt sich schrittweise bis zu einem Minimum (leicht beschleunigend, „modernes“ Gefühl).
  - `ACTION_UP` / `ACTION_CANCEL`: stoppt das automatische Löschen sofort und entfernt alle ausstehenden Handler-Callbacks.
- Alle Handler-Callbacks werden zusätzlich in `onFinishInputView`, `onWindowHidden`, `onStartInputView` und `onDestroy` explizit gestoppt, um Leaks oder Löschen nach Tastatur-/App-Wechsel zu verhindern.
- `delete()` nutzt weiterhin die vorhandene `InputConnection` (`deleteSurroundingTextInCodePoints`/`deleteSurroundingText`) und behandelt zusätzlich:
  - eine aktive Markierung in der Ziel-App (wird zuerst über `getSelectedText`/`commitText("")` gelöscht, bevor auf zeichenweises Löschen zurückgegriffen wird),
  - eine ungültige/fehlende `InputConnection` (kein Absturz),
  - leeren Text (kein Absturz, da `deleteSurroundingText` am Textanfang ein No-Op ist).
- Das Adressfeld (`EditText url`) und der eingebettete Web-Inhalt (`injectBackspaceIntoWeb`) hatten die Markierungs-Logik bereits; daran wurde nichts geändert.

## Keine Auswirkung auf
- Gboard-ähnliches Design, QWERTZ-Layout, restliche Tasten, Emoji/Sonderzeichen/Zahlen, Zwischenablage, Einstellungen.
- Die 🌐-Taste (öffnet weiterhin nur das interne Adressfeld/den internen Browser, keine externe App).
- Bestehende WebView-Lifecycle-Fixes (Pause/Resume beim Ein-/Ausblenden der Tastatur, Wiederherstellung nach `onRenderProcessGone`).
- Gradle/Manifest/GitHub-Actions-Workflow: keine neuen Abhängigkeiten oder Berechtigungen nötig (`Handler`/`Looper` sind Standard-Android-SDK).

## Nachbesserung (Bugfixes)

### Browser-Fenster verschwindet nach „⟳“ und lässt sich nicht wieder öffnen
Ursache: Die vorherige Implementierung hat die WebView komplett aus dem Layout entfernt, zerstört und eine neue Instanz wieder eingehängt. Das Entfernen/Neu-Einhängen einer WebView innerhalb des speziellen Overlay-Fensters eines `InputMethodService` ist unzuverlässig und konnte dazu führen, dass die neue WebView nie sichtbar/aktiv wurde.

Fix: `restartBrowser()` entfernt die WebView jetzt im Normalfall gar nicht mehr aus dem Layout. Stattdessen wird dieselbe Instanz an Ort und Stelle zurückgesetzt:
- laufendes Laden stoppen,
- Navigations-Stack (`clearHistory()`) leeren,
- die zuletzt angezeigte Seite erneut laden — eine echte Navigation erzeugt dabei ohnehin einen komplett neuen JS/DOM-Kontext, Cookies/Login bleiben erhalten.

Nur falls dieser In-Place-Reset tatsächlich eine Exception wirft (kaputter Renderer), wird als Fallback die alte destroy-and-recreate-Logik verwendet — und auch dann wird exakt eine Ersatz-Instanz an derselben Stelle eingesetzt (keine doppelten WebViews).

### Backspace-Wiederholung zu langsam
Die alte Beschleunigungskurve startete bei 400 ms pro Zeichen und brauchte mehrere Sekunden, um auf ihre schnellste Rate (40 ms) zu kommen. Jetzt:
- Auslöseverzögerung vor Beginn der Wiederholung: 350 ms (unverändert spürbar, aber kein Teil der Löschgeschwindigkeit mehr),
- Wiederholung startet bereits bei 90 ms/Zeichen,
- beschleunigt in 8-ms-Schritten bis zu einem Minimum von 20 ms/Zeichen (~50 Zeichen/Sekunde),
- die volle Geschwindigkeit wird dadurch schon nach ca. einer halben Sekunde Halten erreicht statt nach mehreren Sekunden.
