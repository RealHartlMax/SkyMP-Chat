# AGENTS.md — SkyMP Chat Plugin & Slash-Command Hub

## Mission

Implementiere ein **vollständig funktionsfähiges, modulares Ingame-Chat-Plugin** für den Fork:

- Repository: https://github.com/RealHartlMax/skymp
- Zielname: **SkyMP Chat**
- Plugin-ID: `skymp-chat`
- Installationsverzeichnis: `Plugins/SkyMPChat/`
- Sprache der Dokumentation: Deutsch; Code und öffentliche API: konsistentes Englisch.

Das Plugin soll Spieler im Spiel über eine Chat-Oberfläche kommunizieren lassen und **Slash-Commands (`/…`) ausführen, die andere Server-Plugins registrieren**. Der Chat ist eine wiederverwendbare Infrastruktur für spätere RP-Plugins, kein fest verdrahteter Satz von Befehlen. Implementiere die Anwendung tatsächlich; liefere keine bloßen Mockups, TODO-Stubs oder Scheinfunktionen.

## Verbindliche Voranalyse (vor jeder Änderung)

1. Lies `README-plugins.md`, `docs/docs_events_system.md`, `docs/docs_serverside_scripting_reference.md`, `docs/docs_clientside_scripting_reference.md` und `docs/skyrim_platform/browser.md`.
2. Prüfe die tatsächliche Implementierung in `skymp5-server/ts/index.ts` (insbesondere Discovery und `runGamemodePlugin`), `skymp5-client/src/services/services/browserService.ts`, `skymp5-client/src/services/services/gamemodeEventSourceService.ts`, `skymp5-front/src/App.js`, `skymp5-front/src/constructor.js`, `skymp5-front/src/constructorComponents/chat/`, `skymp5-front/src/features/chat/reducer/index.js` und `skymp5-front/docs/chat.md`.
3. Suche nach allen tatsächlich existierenden Chat- und Browser-Transporten sowie Server-Events. Dokumentiere, welche bereits vollständig funktionieren, welche nur UI-Scaffolding sind und wo eine Integrationslücke existiert.
4. Verwende **nur nachweislich verfügbare APIs**. Keine erfundenen `mp.sendChatMessage`, `mp.onChat`, `mp.broadcast` oder `window.skymp`-Semantiken. Der vorhandene Beispiel-Gamemode erwähnt `mp.sendChatMessage` lediglich auskommentiert; das ist kein Beweis für eine verfügbare API.
5. Untersuche die Build- und Deployment-Pfade für `skymp5-front`, SkyrimPlatform/CEF sowie die vom Launcher verteilten Dateien. Ein normales Server-Plugin installiert keine Client-Assets automatisch.
6. Prüfe vorhandene Tests, vorhandene `AGENTS.md`-Dateien und die Repo-Konventionen; überschreibe keine etablierten Anweisungen.
7. Schreibe die verifizierten Funde knapp in `Plugins/SkyMPChat/ARCHITECTURE.md`. Wenn das Checkout gegenüber GitHub inzwischen abweicht, hat die lokale Implementierung Vorrang.

### Bestätigter Stand des Forks als Ausgangspunkt (erneut prüfen)

- `README-plugins.md`: Plugin-Discovery über `Plugins/<Name>/plugin.json`, Standardstatus neuer Plugins deaktiviert, `gamemode` oder `process`, Aktivierung im Admin-Dashboard und wirksam nach Gameserver-Neustart.
- `gamemode`-Plugins werden im Gameserver-Prozess mit globalem `mp` ausgeführt. Der Loader kopiert derzeit **nur den Entry-Inhalt in eine temporäre JS-Datei**; relative `require('./…')`-Aufrufe und ein danebenliegendes `node_modules` sind daher als Laufzeitabhängigkeit unzuverlässig. Entry **vorab als eine CommonJS-Datei bundeln**.
- Ein `process`-Plugin besitzt **kein geteiltes `mp` und keinen automatischen IPC-/RPC-Kanal**. Für diesen Chat daher primär `kind: "gamemode"` verwenden.
- `skymp5-front/src/constructorComponents/chat/index.js` enthält bereits einen React-Chat mit Input, Scrollen, Größenanpassung und Eingabeverlauf; er darf angepasst und wiederverwendet werden.
- `skymp5-front/docs/chat.md` dokumentiert `ADD_CHAT_MSG`, `UPDATE_CHAT_SHOW`, `UPDATE_CHAT_SHOWINPUT` und `cef::chat:send`; die tatsächlich ausführbare Ende-zu-Ende-Verdrahtung muss geprüft werden.
- `browserService.ts` behandelt F6, Enter, Escape und Browser-Fokus. Konflikte mit Inventar, Konsole und Skyrim-Menüs vermeiden.
- `docs/skyrim_platform/browser.md` dokumentiert `browser.executeJavaScript(...)` und `window.skyrimPlatform.sendMessage(...)` sowie den Empfang per `browserMessage`.
- `mp.makeEventSource('_…', sourceCode)` und `mp._eventName = (…) => { … }` sind dokumentierte Wege für Client-zu-Server-Events. Custom-Event-Namen beginnen mit `_`.
- Es gibt **keine automatische per-Plugin-Frontend-Injektion** und keinen integrierten pluginübergreifenden Slash-Command-Registry-Vertrag. Eine kleine gemeinsame Adapter-/Registry-Integration kann erforderlich sein. Diese Änderung explizit und minimiert halten.

## Verifizierter Befund: Warum sich der bisherige Chat möglicherweise nicht öffnen lässt

Bei der Prüfung des GitHub-Standes vom 26.10.2026 wurden folgende **konkrete Integrationslücken** ermittelt. Im lokalen Checkout erneut verifizieren, keine Vermutung als Laufzeittest ausgeben:

1. **Tasten im vorhandenen Client** (`skymp5-client/src/services/services/browserService.ts`): `F2` schaltet `browser.setVisible` um, `F6` schaltet Browser-Fokus um, `Enter` setzt Browser-Fokus auf `true`, `Escape` setzt Fokus auf `false`. Die Tasten öffnen derzeit **kein dediziertes Chatfenster**. Der Browser wird beim Start erst auf unsichtbar und beim ersten Update auf sichtbar gestellt; bestimmte Skyrim-Menüs verbergen ihn. `F2` bedeutet *gesamte CEF-Oberfläche*, nicht *Chat öffnen*.
2. **Rendering-Bedingung** (`skymp5-front/src/App.js`): Der direkte `<Chat />`-Renderpfad befindet sich im Zweig `isLoggined` und wird zusätzlich durch `!isDevUiMode` eingeschränkt. Wenn dieser Zweig nicht aktiv ist, rendert `App` im Widget-Zweig nur tatsächlich gesetzte Widgets. `skymp5-front/src/index.js` initialisiert diese zunächst mit `new Widgets([])`. Ein leerer Widget-Satz führt damit nicht automatisch zu einem sichtbaren Chat. Nicht empfehlen, eine Browser-Dev-Login-Flag als Produktionslösung zu setzen.
3. **Sendeweg im direkten Renderpfad**: `App.js` rendert `<Chat />` ohne `send`-Callback; `constructorComponents/chat/index.js` ruft `send(message)` nur auf, wenn `send !== undefined`. Das Eingabefeld kann daher sichtbar sein, ohne echte Nachrichten zu verschicken. Die Widget-Variante erhält dagegen `widget.send`, sofern solch ein Widget installiert/verdrahtet wird.
4. **Nicht abgesicherte Nachrichtenliste**: Der Chat rendert `window.chatMessages.map(...)` ohne Guard. Prüfe, ob/wann `window.chatMessages` wirklich initialisiert und synchron gehalten wird. Baue eine robuste Default-Liste aus dem vorhandenen Redux-Store/Chat-State ein. Keine globalen Variablen als vorausgesetzt behandeln.
5. **Fokus ist nicht Sichtbarkeit**: `browser.setFocused(true)` beweist weder, dass CEF sichtbar ist, noch, dass das Chat-Element im React-Tree existiert oder dessen Input fokussiert ist. Diese drei Ebenen müssen für das fertiggestellte Plugin explizit zusammenarbeiten.

**Pflicht-Fix für die finale Version:** Ein Spieler mit **frisch gestarteter Installation, ohne gesetzten LocalStorage-Dev-Flags, ohne Testmodus und ohne manuell nachgereichtes Chat-Widget** kann nach Connect den Chat per **Enter** zuverlässig anzeigen und direkt eingeben; **Escape** beendet Chat-Eingabe und gibt Gameplay-Fokus frei. `F6` darf entsprechend der bestehenden Browserfokus-Konvention weiter funktionieren, `F2` bleibt Browser-Sichtbarkeit, sofern ein explizit dokumentierter, rückwärtskompatibler anderer Entschluss technisch erforderlich ist. Bei verstecktem Browser muss Öffnen per Enter ihn wieder sichtbar machen, *ohne* die regulären Skyrim-Menüs zu überlagern. Tasten werden beim Tippen abgefangen, sodass nichts gleichzeitig in Gameplay/Konsole ausgelöst wird. Eine separate, konfigurierbare Chat-Hotkey-Funktion kann ergänzt werden, jedoch nicht als Abhängigkeit für den MVP.

**Verbindliche Funktionstests:**

- [ ] Normale Erstinstallation ohne Login-Flag und ohne vorherige Widgets: Spiel starten, Server verbinden, Chat-HUD existiert, Enter öffnet **sichtbares** Eingabefeld (nicht nur Browser-Fokus).
- [ ] Enter sendet genau eine Nachricht; zweiter Client empfängt sie einmal. Ohne `send`-Callback darf keine vermeintlich erfolgreiche Versendung simuliert werden.
- [ ] Escape schließt Eingabemodus, Eingabetext wird nach definierter Regel erhalten oder verworfen, Tastatureingaben gehen wieder ans Spiel.
- [ ] F6, F2 und Skyrim-Menüs (Inventar, Karte, Konsole usw.) führen nicht zu dauerhaft verborgenem Chat oder gleichzeitigem Gameplay-/Chat-Fokus.
- [ ] Leere/uninitialisierte Chat-Nachrichtenliste stürzt React nicht ab; kein `window.chatMessages.map` auf `undefined`.
- [ ] Chat arbeitet ohne `?devUi=1`, `?admin=1`, `skymp.dev.loggedIn` oder andere Test-Workarounds.
- [ ] Dokumentiere *genau*, welche Taste Chat öffnet, welche Taste Chat fokussiert/schließt und welche den gesamten CEF-Browser umschaltet.

## Architekturregeln

### A. Liefergegenstand: echtes Plugin mit kleinem Integrationsadapter

Primäres Paket:

```text
Plugins/
  SkyMPChat/
    plugin.json
    index.cjs                 # gebundelter, direkt startfähiger Entry
    README.md
    ARCHITECTURE.md
    CHANGELOG.md
    src/                      # wartbare Quellmodule, falls Build nötig
    tests/                    # Unit-/Integrationstests
```

Falls nötig, zusätzliche **kleine** Änderungen am vorhandenen CEF/React-Frontend und am vorhandenen Client-Bridge-Code in den jeweiligen Stammverzeichnissen. Implementiere keinen zweiten Browser, keine zusätzliche öffentliche Chat-Webseite und keinen externen Socket-Server, solange der dokumentierte SkyrimPlatform-/CEF-Transport genügt.

### B. `plugin.json`

Das Manifest muss vom **aktuellen** Validator des Forks akzeptiert werden. Vorgesehener Inhalt:

```json
{
  "name": "skymp-chat",
  "version": "1.0.0",
  "kind": "gamemode",
  "displayName": "SkyMP Chat",
  "description": "Ingame chat and shared slash-command registry for server plugins",
  "main": "index.cjs",
  "optional": false
}
```

`index.cjs` als selbstständiges CommonJS-Bundle für Node erstellen (`esbuild --bundle --platform=node --format=cjs` oder bereits etablierter Repo-Build). Keine Laufzeitannahme, dass eine `src/`-Datei relativ zum temporären Loader-Pfad geladen werden kann. Keine Änderung am globalen Plugin-Loader, um ein Packaging-Problem bequem zu umgehen.

### C. Server ist autoritativ

Datenfluss:

```text
CEF / React Chat Input
   -> vorhandener/schmaler SkyrimPlatform-Browser-Bridge
   -> verifizierter Client-zu-Server-Event/Packet-Weg
   -> SkyMP Chat Plugin: Validierung + Rate Limits + Routing
       |-> normale Nachricht -> Empfängerauswahl -> CEF/React Chat
       `-> /command -> Command Registry -> Fremd-Plugin-Handler
                                    `-> Ergebnis/Fehler nur an berechtigte Empfänger
```

Keine direkte Ausführung von Server- oder Konsolenbefehlen im CEF-Frontend. Vom Client kommen ausschließlich **unvertrauenswürdiger Text und zulässige UI-Metadaten**. Die Spieleridentität (`userId`/Actor-ID) muss der Server aus dem authentifizierten Transportkontext ableiten und darf niemals aus einem vom Client frei gesetzten `playerId`, `role`, `permission` oder `isAdmin` übernommen werden.

### D. Andere Plugins müssen Commands registrieren können

Stelle eine versionierte, dokumentierte und in derselben Gameserver-Prozessinstanz geteilte API bereit, beispielsweise als stabilen Namespace `globalThis.skympChat`, **sofern die Voranalyse den gemeinsamen Prozess bestätigt**. Andere `gamemode`-Plugins nutzen diesen Vertrag ohne Kopieren des Chat-Codes.

Anzustrebender API-Vertrag (Design, kein bereits vorhandener Repo-API-Aufruf):

```ts
interface ChatCommandContext {
  userId: number;
  actorId: number | null;
  playerName: string;
  rawInput: string;
  commandName: string;
  args: string[];
  argsText: string;
  reply(message: string): void;
  error(message: string): void;
}

interface ChatCommandDefinition {
  name: string;               // z. B. "me", ohne führenden Slash
  aliases?: string[];
  description: string;
  usage?: string;             // z. B. "/me <Aktion>"
  permission?: string | null; // immer serverseitig auswerten
  owner: string;              // eindeutige Plugin-ID
  execute(ctx: ChatCommandContext): void | Promise<void>;
}

interface SkyMPChatAPI {
  readonly apiVersion: 1;
  registerCommand(definition: ChatCommandDefinition): void;
  unregisterCommand(owner: string, name: string): void;
  getCommandsForPlayer(userId: number): Array<{
    name: string; description: string; usage?: string;
  }>;
  sendToPlayer(userId: number, message: string, category?: string): void;
  broadcast(message: string, category?: string): void;
}
```

Implementierungshinweise:

- Registrierungen aus anderen Plugins bleiben voneinander isoliert. Speicherung über `Map`/Registry mit `owner`, kanonischem Namen, Alias-Auflösung und definiertem Lebenszyklus.
- Doppelte Namen/Aliase **nicht stillschweigend überschreiben**. Fehler mit Eigentümer-IDs protokollieren; registrierende Plugins erhalten einen nachvollziehbaren Fehler.
- `registerCommand` soll vorhersagbar funktionieren, sobald SkyMP Chat geladen ist. Lade SkyMP Chat vor Commands anbietenden Plugins (`pluginsLoadOrder`), und dokumentiere das. Falls Plugins vorher geladen werden können, einen minimalen, dokumentierten Wartemechanismus vorsehen; keine unendlichen Polling-Loops.
- Names und Aliases mit klarer erlaubter Syntax validieren (z. B. ASCII-Buchstaben, Ziffern, `-`, `_`, Maximalgröße). Commands bei der Auflösung case-insensitive behandeln.
- Kein blindes `eval`, `new Function`, `child_process.exec` oder Konsolen-Befehl anhand des Chat-Textes; nur registrierte Handler ausführen.
- Jede Ausführung durch `try/catch` bzw. `await` absichern. Unbehandelte Plugin-Fehler dürfen den Gameserver nicht abstürzen lassen.
- Permissions durch pluggable, **serverseitige** Permission-Resolver oder die tatsächlich vorhandene Server-Rollen-/Admin-API prüfen. Ohne verifizierte Freigabe **deny by default** für geschützte Commands. Ein fehlender Permission-Resolver darf Rechte niemals automatisch gewähren.
- Optional Command-Metadaten für Parameterhinweise und Vorschlagsliste verfügbar machen; sensible Admin-Befehle nicht für unberechtigte Nutzer ausgeben.
- Lifecycle und Neustarts berücksichtigen: Registry nur im laufenden Prozess; beim vollständigen Neustart werden Commands durch aktive Plugins neu registriert. Keine falsche Behauptung von Hot-Reload-Support.

**Zwingender Integrationstest:** Ein zweites Minimal-Plugin muss `/ping` registrieren können, ohne SkyMP Chat intern zu verändern. Bei Eingabe `/ping` bekommt der Absender `Pong!`. Ein Beispiel-Plugin dafür mitliefern oder als Testfixture bereitstellen.

### E. Slash-Command-Parsing

Eingabe auf dem Server nach Validierung klassifizieren:

- `Hallo zusammen` -> normale Chat-Nachricht.
- `/ping` -> registrierter Befehl `ping`, Argumentliste leer.
- `/tp Spieler 10` -> Name `tp`, `args = ["Spieler", "10"]`, `argsText = "Spieler 10"`.
- `/me zieht den Hut` -> ein fremdes RP-Plugin darf `me` registrieren.
- `//Beispiel` -> als literal `/Beispiel` senden (Slash-Escape; dokumentieren und testen).
- `/` oder unbekannte Commands -> hilfreiche private Rückmeldung, keine globale Anzeige.

Parserregeln: Führender Slash **nur am Anfang** des Eingabetexts nach bewusst definierter Whitespace-Behandlung; `/` mitten in normalem Text ist kein Command. Argumente dürfen Leerzeichen und optional Zitierzeichen/Escape-Zeichen enthalten. Für `argsText` den Originalrest korrekt bewahren; keine unerwartete Kleinschreibung der Argumente. Leere, zu lange oder fehlerhafte Eingaben behandeln. Slash-Commands dürfen nicht als öffentliche Chat-Nachrichten erscheinen, außer der jeweilige Handler veröffentlicht explizit eine formatierte RP-Nachricht.

Core-Commands des Chat-Plugins: `/help` und `/commands` (sichtbare Commands + Beschreibung), optional `/clear` nur clientseitig für lokale Anzeige. Keine fremden RP-Commands wie `/me`, `/ooc`, `/w`, `/tp` fest im Chat-Kern verbauen. Diese gehören in eigenständige Plugins, sofern der Operator das möchte.

### F. Chatnachrichten

MVP:

1. Spieler kann über die bestehende Ingame-CEF-Oberfläche Text eingeben und per Enter senden.
2. Jede akzeptierte Nachricht erscheint beim passenden Empfängerkreis mit vertrauenswürdigem Anzeigenamen und Zeitstempel (Zeit vom Server, Anzeige lokal formatierbar).
3. System-, Spieler-, Fehler- und Command-Antworten visuell unterscheiden; Fehler und sensible Command-Ausgaben standardmäßig **privat** an den Ausführenden.
4. Standardkanal `global` für das MVP; API auf künftige Kanäle erweiterbar. Optional `local`/Proximity nur implementieren, wenn Distanz und Welt-/Zellenprüfung tatsächlich korrekt serverseitig möglich und getestet sind.
5. Nachrichtenlimits und Unicode: Defaults z. B. max. 500 Zeichen pro Nachricht, definierte Häufigkeitsgrenzen; serverseitig validieren. Die UI darf ein komfortables höheres Eingabelimit nur dann verwenden, wenn beide Limits konsistent sind.
6. Nachrichtentext **sicher als Text rendern**, niemals ungefiltertes `innerHTML` aus Spielertext. Keine beliebigen CSS-Farben, HTML, Scripts oder Event-Attribute aus Chat-Eingaben übernehmen. Vorhandene alte Farbmarkup-Konventionen nur mit abgesicherter Parsing-Logik unterstützen.
7. Scrollback als begrenzter Ringpuffer (z. B. 100 Nachrichten) ohne ungebremstes Speicherwachstum. Neue Nachrichten nur automatisch nach unten scrollen, wenn der Nutzer bereits nahe dem Ende war.
8. Keine persistente Speicherung privater Chats per Default. Falls Audit-/Moderationslogs erforderlich werden, getrennte, ausdrücklich konfigurierbare Option mit Datenschutzhinweis vorsehen.
9. Saubere Trennung von Nachrichteneingang, Befehlsantwort und Renderformat, damit spätere Moderation/Filter/Channels möglich sind.

### G. UI/UX im vorhandenen React-Chat

**Bestehenden Code wiederverwenden**, insbesondere `skymp5-front/src/constructorComponents/chat/`, Redux-Chat-State und vorhandene Browserfokus-Mechanik. Es darf keinen zweiten unverbundenen Chat-Store geben.

- Öffnen/Eingabe bevorzugt über bestehende Enter-/F6-Bindings; tatsächliche Bindings testen, keine Doppelregistrierung. Escape schließt Fokus oder Eingabe entsprechend bestehendem Verhalten. F2 darf nicht inkonsistent mit vorhandener Browser-Sichtbarkeit werden.
- Chatnachrichten, Spielername, Zeit, Systemmeldungen und Ergebnisse der Commands anzeigen.
- Wenn `/` getippt wird: optionale Command-Autovervollständigung aus **berechtigten, registrierten** Commands; Tab/Up/Down/Enter konfliktfrei behandeln. Bei fehlender Vorschlagsliste trotzdem korrekte Serverausführung.
- Vorhandenen Eingabeverlauf erhalten; sinnvolles Keyboard-Verhalten bei geöffnetem Inventar, Konsole, Dialogen und Chat-Fokus sicherstellen.
- Chat-HUD optisch Skyrim-/RP-kompatibel, gut lesbar, transparent und konfigurierbar (Fontgröße, Transparenz, ggf. Position).
- Keyboardfokus beim Öffnen/Schließen korrekt setzen, damit Spieler nicht gleichzeitig in Skyrim und im Eingabefeld Aktionen auslösen.
- Verbindungstrennung, Server-Neustart und fehlendes Chat-Plugin sichtbar und fehlertolerant behandeln; keine endlosen Client-Fehlerschleifen.
- Kein hart codierter Testmodus als Produktionsweg. Falls bestehender Chat zwar gerendert wird, aber `send` nicht durchgereicht wird, die echte Anbindung reparieren und testen.
- Deutsche UI-Texte als Standard oder über vorhandenes i18n-System pflegen; mindestens Deutsch/Englisch, sofern Repo-I18n einheitlich integriert ist.

### H. Browser-/Netzwerk-Bridge

Bevorzugte Implementierung: bestehendes Chat-Send-Event oder `window.skyrimPlatform.sendMessage` -> `browserMessage` -> **verifizierter** `mp.makeEventSource`/Netzwerkpfad -> Serverhandler. Eingehende Chat-Nachrichten über die tatsächlich verfügbare Owner-Property, Clientcode-Ausführung oder Action/Redux-Bridge an die UI übertragen. Kein erfundener generischer Eventbus.

- Namespaced Nachrichten verwenden (`skympChat:*`) und nicht mit `front-loaded`, `onInput` oder anderen Browser-Nachrichten kollidieren.
- Event-Listener nur einmal binden und bei Lebenszyklusänderungen korrekt bereinigen. Wiederholtes Frontend-Laden darf keine Nachrichten duplizieren.
- Payloads strikt begrenzen, typisieren und validieren. Keine unnötigen Felder, insbesondere keine Client-Rollen-/Admin-Flags.
- Bereits vorhandene Nachrichten-/State-Pfade auf Kompatibilität untersuchen. Der Chat darf bestehende Systeme wie `SkillsMenu`, `AdminDashboard`, `AnimList` oder `ServerList` nicht beschädigen.
- Nachweis, dass Nachricht tatsächlich Server -> anderer Client und nicht nur im sendenden Browser funktioniert.

## Sicherheit und Robustheit

- Keine Befehlsrechte aufgrund von UI-Sichtbarkeit oder unverifizierten Daten.
- Schutz gegen Spam/Flood und Burst (Token-Bucket oder Sliding Window), getrennte Limits für Chat und Commands; Admin-Bypass nur wenn explizit serverseitig autorisiert.
- Nachrichten, Fehler und Debug-Logs nicht unkontrolliert mit Zugangsdaten oder privaten Payloads anreichern.
- Command-Ausführung pro Spieler fair begrenzen, asynchrone Handler mit kontrollierter Fehlerbehandlung; keine unendliche Queue.
- Plugin-Abwesenheit/Deaktivierung darf Gameplay und Verbindung nicht brechen.
- Konflikte mehrerer Plugins eindeutig berichten statt stillem Override.
- Zugänglichkeit: tastaturbedienbar, Fokusindikator, Kontrast, skalierbare Schrift.

## Tests / Abnahmekriterien

Baue möglichst automatische Tests und einen manuellen Ingame-Smoke-Test. Als bestanden gilt:

- [ ] `Plugins/SkyMPChat/plugin.json` wird vom echten Loader erkannt; neues Plugin ist im `safe`-Modus zunächst deaktiviert.
- [ ] Nach Aktivierung und vollständigem Gameserver-Neustart wird das Plugin geladen; kein zusätzlicher Node-/Web-Server erforderlich.
- [ ] Zwei getrennte Clients: A sendet `Hallo`, A und B empfangen die Nachricht genau einmal; Name und Text stimmen.
- [ ] Slash-Command `/ping` des mitgelieferten **Fremd-Plugins** liefert ausschließlich an A `Pong!`; keine Chatveröffentlichung des rohen Befehls.
- [ ] `/help` listet nur für A autorisierte Commands; Rechte werden serverseitig geprüft.
- [ ] Ein nicht autorisierter Befehl führt **keine** Aktion aus und liefert private Ablehnung.
- [ ] Unbekannter Befehl, `//escaped`, Whitespace, Quotes und leere Eingabe sind getestet.
- [ ] Gleiche Command-Namen/Aliase aus zwei Plugins erzeugen einen kontrollierten Konflikt.
- [ ] HTML-/Script-Injection wird als harmloser Text angezeigt, kein XSS.
- [ ] Flooding löst definierte Begrenzung aus; normale Nutzer können nach Abklingzeit weitersenden.
- [ ] Chat öffnen/schließen, Eingabe, Scrollback und Skyrim-Menüs funktionieren ohne Eingabekonflikt.
- [ ] UI-Reload/Client-Reconnect produziert keine doppelten Handler.
- [ ] Plugin deaktivieren + Server neu starten: kein defekter UI-/Netzwerkzustand.
- [ ] Bestehende Server-Plugin- und Frontend-Tests bleiben grün; dokumentiere die exakten ausgeführten Kommandos.

Referenz-Tests im Repo vor Ausführung prüfen: `skymp5-server/tests/plugins.integration.mjs`, Frontend-Skripte aus `skymp5-front/package.json`. Tests niemals als bestanden deklarieren, wenn sie nicht tatsächlich liefen.

## Vorgehensweise für den Coding-Agenten

**Phase 1 — Audit:** Istzustand belegen; Datenfluss Frontend/Client/Server aufzeichnen; genaue Integration festlegen; Risiken dokumentieren.

**Phase 2 — MVP-Server:** Bundle-fähigen `gamemode`-Plugin-Entry, Chat-Routing, validierte Nachrichten, Basic Flood-Control, Registry und `/help` bauen. Tests zuerst oder parallel.

**Phase 3 — Client-Brücke und UI:** bestehenden React-Chat funktional anschließen; CEF-Fokus und Chat-Events robust machen; neue Paket-/Launcher-Dateien nur hinzufügen, wenn der vorhandene Auslieferungsweg das nachweislich erfordert.

**Phase 4 — Erweiterbarkeit:** Beispiel-Zweitplugin mit `/ping`, Owner-/Alias-/Permissions-Tests, Hilfe/Autocomplete, verständliche Public-API-Docs.

**Phase 5 — Integration & Abschluss:** Build, Tests, mindestens dokumentierter Zwei-Client-Smoke-Test, Installation, Startreihenfolge und notwendige Neustarts; Dateiübersicht und alle verbleibenden Einschränkungen nennen.

### Nicht erlaubt

- Keine unnötige Komplettneuschreibung der UI oder des Server-Plugin-Loaders.
- Kein Kopieren eines bereits vorhandenen Chat-Features in eine zweite Parallelimplementierung.
- Kein blindes Herunterladen oder Ausführen externer Inhalte.
- Keine unbelegte Behauptung, dass ein Server-Plugin Clientdateien automatisch verteilt.
- Kein Hardcoding von RP-Befehlen fremder Plugins in den Chat-Kern.
- Keine Änderungen an Launcher-/Update-Routinen ohne nachgewiesenen Integrationsbedarf.
- Keine Änderungen an bestehenden Benutzer-/Adminrechten ohne vorherige Prüfung.

## Auszuliefernde Dokumentation

- `Plugins/SkyMPChat/README.md`: Installation als entpackter Plugin-Ordner, Aktivierung über Ressourcen -> Server-Plugins, Neustart, Konfiguration, UI-Bedienung, Features.
- `Plugins/SkyMPChat/ARCHITECTURE.md`: echte Transportwege, vorhandene und neue Hooks, begründete Core-/Client-Anpassungen.
- `Plugins/SkyMPChat/COMMAND_API.md`: API-Version, `registerCommand`, `unregisterCommand`, Context/Permission-Resolver, Codebeispiel und Migration.
- `Plugins/SkyMPChat/CHANGELOG.md`: Änderungen und Kompatibilität.
- Beispiel-Plugin/-Fixture für `/ping` mit eigenem `plugin.json` und funktionierendem Build/Entry.
- Konkrete Installations- und Build-Befehle für Windows und Linux, soweit getestet.

## Abschlussbericht des Coding-Agenten

Berichte nach Umsetzung knapp und überprüfbar:

1. **Istzustand:** Welche vorhandenen Chat-/CEF-Teile wurden benutzt?
2. **Änderungen:** Welche Plugin-, Core-, Client- und UI-Dateien wurden verändert und warum?
3. **Benutzung:** Wie öffnet ein Spieler den Chat, und wie registriert ein zweites Plugin `/ping`?
4. **Tests:** Exakte Befehle, Pass/Fail, was nicht ausführbar war.
5. **Deployment:** Was müssen Gameserver, Launcher und Client bekommen? Welche Neustarts sind zwingend?
6. **Restpunkte:** Welche Einschränkungen oder Risiken sind offen?

**Priorität:** Ein verlässlich funktionierendes, kleines MVP mit echter Plugin-Interop ist wichtiger als viele unfertige Features. Beende den Auftrag erst mit überprüfbar gesendeten Chatnachrichten und einem erfolgreich vom Fremd-Plugin ausgeführten Slash-Command oder kennzeichne präzise, welcher Integrationsschritt mangels Testumgebung nicht verifiziert werden konnte.
