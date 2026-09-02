# Projektdokumentation: Simulation eines OCPP-Ladeinfrastruktur-Backends (CSMS)

**Ziel des Projekts:** 
Aufbau eines funktionsfähigen Charge Point Management Systems (CSMS) zur Verwaltung von Ladesäulen, Nutzern und RFID-Ladekarten. Anschließend wird ein kompletter Ladezyklus einer Wallbox über das Open Charge Point Protocol (OCPP 1.6J) via WebSockets simuliert.

**Verwendete Technologien:**
*   **SteVe:** Open-Source OCPP-Server (Backend)
*   **GitHub Codespaces:** Cloud-basierte Docker-Umgebung (umgeht lokale Admin-Restriktionen)
*   **Postman:** API-Client zur WebSocket-Simulation (simuliert die Wallbox)

---

## Phase 1: Backend-Deployment (Server starten)

Da für lokale Docker-Installationen oft Administrator-Rechte fehlen, wird das System in einer temporären Cloud-Umgebung gehostet.

1. Das offizielle GitHub-Repository öffnen: `https://github.com/steve-community/steve`
2. Auf den grünen Button **Code** klicken, den Reiter **Codespaces** wählen und einen neuen Codespace erstellen (`+`).
3. Sobald der webbasierte Editor geladen ist, in das integrierte Terminal am unteren Bildschirmrand klicken.
4. Den Server und die Datenbank über Docker starten:
   ```bash
   docker-compose up -d
   ```
5. Im Reiter **Ports** (neben dem Terminal) den Port `8180` suchen, per Rechtsklick die **Port Visibility** auf **Public** stellen.
6. Auf das Weltkugel-Symbol klicken, um die Webadresse zu öffnen. An die URL `/steve/manager/home` anhängen.
   *(Login-Daten: `admin` / `1234`)*

---

## Phase 2: Systemkonfiguration (Stammdaten anlegen)

Im SteVe-Webinterface (der Ansicht für den Charge Point Operator) müssen nun die Infrastruktur und die Berechtigungen eingerichtet werden.

1. **Wallbox anlegen:**
   * Navigation: *Data Management -> Charge Points -> Add*
   * *ChargeBox ID:* `Demo-Wallbox-1` eintragen und speichern.
2. **Nutzer anlegen:**
   * Navigation: *Data Management -> Users -> Add*
   * Daten eintragen (z.B. Vorname: Max, Nachname: Mustermann) und speichern.
3. **RFID-Ladekarte verknüpfen:**
   * Navigation: *Data Management -> OCPP Tags -> Add*
   * *Tag ID:* `DEADBEEF` (Simulierte Seriennummer der Ladekarte)
   * *Parent ID:* Den zuvor erstellten Nutzer (Max Mustermann) aus dem Dropdown wählen und speichern.

---

## Phase 3: Client-Konfiguration (Postman als Wallbox)

Die Wallbox kommuniziert nicht über normales HTTP, sondern über eine dauerhafte WebSocket-Verbindung. Postman übernimmt hier die Rolle der Ladesäule.

1. In Postman einen neuen **WebSocket Request** erstellen (Nicht HTTP!).
2. In den Tab **Headers** wechseln und zwingend folgende Information hinzufügen, damit der Server das Protokoll akzeptiert:
   * **Key:** `Sec-WebSocket-Protocol`
   * **Value:** `ocpp1.6`
3. Die Server-URL in die Adresszeile eintragen. (Wichtig: Das Präfix `wss://` nutzen und die ChargeBox ID anhängen):
   `wss://<DEINE-CODESPACE-URL>/steve/websocket/CentralSystemService/Demo-Wallbox-1`
4. Auf **Connect** klicken. (Unten in der Konsole muss `Connected` erscheinen).

---

## Phase 4: Die OCPP-Simulation (Der Ladezyklus)

OCPP-Nachrichten sind JSON-Arrays nach dem Schema: `[Nachrichtentyp, Nachrichten-ID, Befehl, Payload/Daten]`.
Die folgenden Blöcke nacheinander in das **Message**-Feld von Postman kopieren und jeweils auf **Send** klicken.

### Schritt 1: BootNotification (Wallbox schaltet sich ein)
Die Säule meldet sich beim Backend als online und funktionsfähig.
```json
[2, "msg-01", "BootNotification", {"chargePointVendor": "Postman", "chargePointModel": "Simulator"}]
```
*Erwartete Server-Antwort:* `Accepted`

### Schritt 2: Authorize (Nutzer hält RFID-Karte vor)
Die Säule fragt das Backend, ob die vorgehaltene Karte laden darf.
```json
[2, "msg-02", "Authorize", {"idTag": "DEADBEEF"}]
```
*Erwartete Server-Antwort:* `Accepted` (Da die Karte in Phase 2 angelegt wurde).

### Schritt 3: StartTransaction (Ladevorgang beginnt)
Die Freigabe war erfolgreich, das Relais schaltet und der Strom fließt.
*(Hinweis: Der Zeitstempel wird bewusst in die Vergangenheit gesetzt z.B. 10:00 Uhr UTC, um Zeitzonen-Konflikte im Filter zu vermeiden).*
```json
[2, "msg-03", "StartTransaction", {"connectorId": 1, "idTag": "DEADBEEF", "meterStart": 0, "timestamp": "2026-09-02T10:00:00Z"}]
```
*Erwartete Server-Antwort:* Der Server vergibt eine **transactionId** (z.B. `1`). Diese muss für den Stop-Befehl notiert werden.

### Schritt 4: StopTransaction (Ladekabel wird abgezogen)
Der Ladevorgang wird beendet und der finale Zählerstand an das Backend übermittelt.
```json
[2, "msg-04", "StopTransaction", {"transactionId": 1, "meterStop": 15000, "timestamp": "2026-09-02T11:00:00Z"}]
```
*(Es wurden fiktive 15.000 Wh = 15 kWh geladen).*

---

## Phase 5: Auswertung (Das Resultat)

Um den Erfolg der Simulation zu beweisen, wechselt man zurück in das SteVe-Webinterface.

1. Navigation: *Operations -> Transactions*
2. **Filter anpassen:** SteVe filtert in der Standardansicht streng nach aktuellen Zeitzonen. Um sicherzugehen, dass beendete Transaktionen sichtbar sind, das **End Date** in der Suchmaske auf den morgigen Tag setzen und auf *Search* klicken.
3. **Ergebnis:** Die simulierte Transaktion wird mitsamt Ladedauer, Startzeitpunkt, geladener Energie (15 kWh) und der RFID-Karte (`DEADBEEF`) sauber in der Datenbank protokolliert. Dies bildet die Grundlage für eine spätere, automatisierte Rechnungsstellung an den Nutzer "Max Mustermann".