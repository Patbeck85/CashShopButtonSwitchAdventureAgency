# Vollständige Implementierung: Cash Shop zu Adventure Agency Umleitung (rAthena)

## Übersicht
Diese Implementierung ermöglicht es Administratoren, den Cash Shop Button serverseitig zur Adventure Agency umzuleiten mit einer dynamischen Umschaltfunktion.

## 📋 Farbcode-Legende
- 🟢 **Grün/Fett** = Code der hinzugefügt werden muss
- ⚪ Grau/Normal = Bestehender Code (zur Orientierung)
- 📝 Kursiv = Kommentare und Erklärungen

---

## 📌 SCHNELLÜBERSICHT - Was muss wo hinzugefügt werden?

| Datei | Was hinzufügen | Beschreibung |
|-------|---------------|--------------|
| **src/map/battle.hpp** | `int cashshop_redirect_to_adventure;` | Variable im struct deklarieren |
| **src/map/battle.cpp** | `0,` bei Initialisierung | Variable mit 0 initialisieren |
| **src/map/battle.cpp** | `{ "cashshop_redirect_to_adventure", ... }` | Config-Eintrag im Array |
| **src/map/clif.cpp** | `if (battle_config.cash...) { ... }` | Umleitung in clif_parse_CashShopOpen |
| **src/map/clif.cpp** | `void clif_adventureagency_open(...)` | Neue Funktion hinzufügen |
| **src/map/clif.hpp** | `void clif_adventureagency_open(...);` | Funktionsdeklaration |
| **src/map/atcommand.cpp** | `ACMD(cashredirect) { ... }` | Komplette Command-Funktion |
| **src/map/atcommand.cpp** | `ACMD_DEF(cashredirect),` | Command registrieren |
| **conf/battle/client.conf** | `cashshop_redirect_to_adventure: 0` | Config-Option |

---

## Schritt 1: Battle Config Header erweitern

**📁 Datei: `src/map/battle.hpp`**

Suchen Sie nach anderen `int` Deklarationen im `battle_config` struct und fügen Sie hinzu:

```cpp
struct Battle_Config {
    // ... bestehende Variablen ...
    
    int cashshop_redirect_to_adventure;  // ✅ NEU: Diese Zeile hinzufügen
    
    // ... rest des structs ...
};
```

### ✅ ZU ERGÄNZEN:
```cpp
int cashshop_redirect_to_adventure;
```

---

## Schritt 2: Battle Config Implementierung

**📁 Datei: `src/map/battle.cpp`**

### 2.1: Variable initialisieren

Suchen Sie die `static struct Battle_Config battle_config` Initialisierung und fügen Sie hinzu:

```cpp
static struct Battle_Config battle_config = {
    // ... bestehende Werte ...
    
    0, // ✅ NEU: cashshop_redirect_to_adventure (Standard: deaktiviert)
    
    // ... rest der Werte ...
};
```

### ✅ ZU ERGÄNZEN:
```cpp
0, // cashshop_redirect_to_adventure
```

---

### 2.2: Config-Option registrieren

Suchen Sie das `config_data[]` Array (normalerweise gegen Ende der Datei) und fügen Sie hinzu:

```cpp
static const struct _battle_data {
    const char *str;
    int *val;
    int defval;
    int min;
    int max;
} battle_data[] = {
    // ... bestehende Einträge ...
    
    // ✅ NEU: Diese komplette Zeile hinzufügen
    { "cashshop_redirect_to_adventure",     &battle_config.cashshop_redirect_to_adventure,      0,      0,      1       },
    
    // ... rest der Einträge ...
};
```

### ✅ ZU ERGÄNZEN:
```cpp
{ "cashshop_redirect_to_adventure",     &battle_config.cashshop_redirect_to_adventure,      0,      0,      1       },
```

---

## Schritt 3: Packet Handler modifizieren

**📁 Datei: `src/map/clif.cpp`**

### 3.1: Cash Shop Open Packet abfangen

Suchen Sie die Funktion `clif_parse_CashShopOpen` (oder ähnlich benannt, je nach rAthena Version):

```cpp
/// Request to open cash shop (CZ_SE_CASHSHOP_OPEN).
/// 0844
void clif_parse_CashShopOpen(int fd, struct map_session_data *sd) {
    nullpo_retv(sd);
    
    // ========== ✅ NEU: ANFANG - Diesen kompletten Block hinzufügen ==========
    // Prüfen ob Umleitung zur Adventure Agency aktiv ist
    if (battle_config.cashshop_redirect_to_adventure) {
        // Öffne stattdessen Adventure Agency
        clif_adventureagency_open(sd);
        return;
    }
    // ========== ✅ NEU: ENDE ==========
    
    // Originaler Cash Shop Code (unverändert lassen)
    if (battle_config.feature_roulette == 0 || sd->state.block_action & PCBLOCK_CASH) {
        clif_messagecolor(&sd->bl, color_table[COLOR_RED], msg_txt(sd, 723), false, SELF);
        return;
    }
    
    // ... rest des originalen Codes ...
}
```

### ✅ ZU ERGÄNZEN (direkt nach `nullpo_retv(sd);`):
```cpp
// Prüfen ob Umleitung zur Adventure Agency aktiv ist
if (battle_config.cashshop_redirect_to_adventure) {
    // Öffne stattdessen Adventure Agency
    clif_adventureagency_open(sd);
    return;
}
```

---

### 3.2: Adventure Agency Funktion erstellen

**📁 Datei: `src/map/clif.cpp`**

Falls `clif_adventureagency_open` in Ihrer Version nicht existiert, müssen Sie es erstellen.
Fügen Sie diese komplette Funktion irgendwo in der Datei hinzu (z.B. bei anderen clif_ Funktionen):

```cpp
// ========== ✅ NEU: KOMPLETTE FUNKTION HINZUFÜGEN ==========
/// Öffnet die Adventure Agency für den Spieler
void clif_adventureagency_open(struct map_session_data *sd) {
    int fd;
    
    nullpo_retv(sd);
    fd = sd->fd;
    
#if PACKETVER >= 20111122
    WFIFOHEAD(fd, packet_len(0x97f1));
    WFIFOW(fd, 0) = 0x97f1;  // Adventure Agency Packet
    WFIFOSET(fd, packet_len(0x97f1));
#else
    clif_messagecolor(&sd->bl, color_table[COLOR_RED], "Adventure Agency ist in diesem Client nicht verfügbar.", false, SELF);
#endif
}
// ========== ✅ NEU: ENDE ==========
```

### ✅ KOMPLETTE FUNKTION ZU ERGÄNZEN:
```cpp
void clif_adventureagency_open(struct map_session_data *sd) {
    int fd;
    
    nullpo_retv(sd);
    fd = sd->fd;
    
#if PACKETVER >= 20111122
    WFIFOHEAD(fd, packet_len(0x97f1));
    WFIFOW(fd, 0) = 0x97f1;
    WFIFOSET(fd, packet_len(0x97f1));
#else
    clif_messagecolor(&sd->bl, color_table[COLOR_RED], "Adventure Agency ist in diesem Client nicht verfügbar.", false, SELF);
#endif
}
```

---

### 3.3: Funktionsdeklaration im Header

**📁 Datei: `src/map/clif.hpp`**

Fügen Sie die Funktionsdeklaration hinzu (suchen Sie nach anderen `void clif_` Deklarationen):

```cpp
// ... andere clif_ Funktionen ...

void clif_adventureagency_open(struct map_session_data *sd);  // ✅ NEU: Diese Zeile hinzufügen

// ... rest der Deklarationen ...
```

### ✅ ZU ERGÄNZEN:
```cpp
void clif_adventureagency_open(struct map_session_data *sd);
```

---

## Schritt 4: Admin Command erstellen

**📁 Datei: `src/map/atcommand.cpp`**

### 4.1: Command Funktion implementieren

Suchen Sie nach anderen `ACMD` Funktionen und fügen Sie diese **KOMPLETTE FUNKTION** hinzu:

```cpp
// ========== ✅ NEU: KOMPLETTE FUNKTION HINZUFÜGEN ==========
/*==========================================
 * @cashredirect - Schaltet Cash Shop Umleitung zur Adventure Agency um
 *------------------------------------------*/
ACMD(cashredirect) {
    if (battle_config.cashshop_redirect_to_adventure == 0) {
        battle_config.cashshop_redirect_to_adventure = 1;
        clif_displaymessage(fd, "Cash Shop Button öffnet jetzt die Adventure Agency.");
        ShowInfo("Admin %s hat Cash Shop Umleitung aktiviert.\n", sd->status.name);
    } else {
        battle_config.cashshop_redirect_to_adventure = 0;
        clif_displaymessage(fd, "Cash Shop Umleitung deaktiviert. Normaler Cash Shop wird geöffnet.");
        ShowInfo("Admin %s hat Cash Shop Umleitung deaktiviert.\n", sd->status.name);
    }
    
    return true;
}
// ========== ✅ NEU: ENDE ==========
```

### ✅ KOMPLETTE FUNKTION ZU ERGÄNZEN:
```cpp
ACMD(cashredirect) {
    if (battle_config.cashshop_redirect_to_adventure == 0) {
        battle_config.cashshop_redirect_to_adventure = 1;
        clif_displaymessage(fd, "Cash Shop Button öffnet jetzt die Adventure Agency.");
        ShowInfo("Admin %s hat Cash Shop Umleitung aktiviert.\n", sd->status.name);
    } else {
        battle_config.cashshop_redirect_to_adventure = 0;
        clif_displaymessage(fd, "Cash Shop Umleitung deaktiviert. Normaler Cash Shop wird geöffnet.");
        ShowInfo("Admin %s hat Cash Shop Umleitung deaktiviert.\n", sd->status.name);
    }
    return true;
}
```

---

### 4.2: Command registrieren

**📁 Datei: `src/map/atcommand.cpp`**

Suchen Sie das `AtCommandInfo atcommand_info[]` Array und fügen Sie hinzu:

```cpp
static AtCommandInfo atcommand_info[] = {
    // ... bestehende Commands ...
    
    ACMD_DEF(cashredirect),  // ✅ NEU: Diese Zeile hinzufügen
    
    // ... rest der Commands ...
};
```

### ✅ ZU ERGÄNZEN:
```cpp
ACMD_DEF(cashredirect),
```

---

## Schritt 5: Config-Datei erstellen/erweitern

**📁 Datei: `conf/battle/client.conf`**

Fügen Sie am Ende oder in einem passenden Bereich diese **KOMPLETTEN ZEILEN** hinzu:

```conf
// ========== ✅ NEU: KOMPLETTEN BLOCK HINZUFÜGEN ==========
//--------------------------------------------------------------
// Cash Shop Einstellungen
//--------------------------------------------------------------

// Cash Shop Button zur Adventure Agency umleiten?
// 0: Normaler Cash Shop (Standard)
// 1: Adventure Agency stattdessen öffnen
// Hinweis: Kann zur Laufzeit mit @cashredirect umgeschaltet werden
cashshop_redirect_to_adventure: 0
// ========== ✅ NEU: ENDE ==========
```

### ✅ KOMPLETT ZU ERGÄNZEN:
```conf
//--------------------------------------------------------------
// Cash Shop Einstellungen
//--------------------------------------------------------------

// Cash Shop Button zur Adventure Agency umleiten?
// 0: Normaler Cash Shop (Standard)
// 1: Adventure Agency stattdessen öffnen
// Hinweis: Kann zur Laufzeit mit @cashredirect umgeschaltet werden
cashshop_redirect_to_adventure: 0
```

---

## Schritt 6: Erweiterte Version mit Permission Check (Optional)

**📁 Datei: `conf/groups.conf`**

Falls Sie möchten, dass nur bestimmte GM-Level den Befehl nutzen können:

```yaml
// ========== ✅ OPTIONAL: In der entsprechenden Gruppe hinzufügen ==========
// In der entsprechenden Gruppe (z.B. Admin mit ID 99)
{
    id: 99
    name: "Admin"
    level: 99
    commands: {
        // ... andere Commands ...
        cashredirect: true  // ✅ NEU: Diese Zeile hinzufügen
    }
}
// ========== ✅ ENDE ==========
```

### ✅ OPTIONAL ZU ERGÄNZEN:
```yaml
cashredirect: true
```

---

**📁 Datei: `src/map/atcommand.cpp`**

Und passen Sie den atcommand Code an (Permission Check):

```cpp
ACMD(cashredirect) {
    // ========== ✅ OPTIONAL: Permission Check hinzufügen ==========
    // Permission Check (optional)
    if (pc_get_group_level(sd) < 90) {
        clif_displaymessage(fd, "Sie haben keine Berechtigung für diesen Befehl.");
        return false;
    }
    // ========== ✅ ENDE ==========
    
    // ... rest des Codes wie oben ...
}
```

---

## Schritt 7: Erweiterte Features (Optional)

### 7.1: Status-Abfrage hinzufügen

**📁 Datei: `src/map/atcommand.cpp`**

Ersetzen Sie die einfache `ACMD(cashredirect)` Funktion mit dieser erweiterten Version:

```cpp
// ========== ✅ OPTIONAL: ERWEITERTE VERSION MIT STATUS-ABFRAGE ==========
ACMD(cashredirect) {
    if (!message || !*message || (strcmp(message, "on") != 0 && strcmp(message, "off") != 0 && strcmp(message, "status") != 0)) {
        clif_displaymessage(fd, "Verwendung: @cashredirect <on|off|status>");
        return false;
    }
    
    if (strcmp(message, "status") == 0) {
        if (battle_config.cashshop_redirect_to_adventure) {
            clif_displaymessage(fd, "Cash Shop Umleitung: AKTIV (Adventure Agency wird geöffnet)");
        } else {
            clif_displaymessage(fd, "Cash Shop Umleitung: INAKTIV (Normaler Cash Shop)");
        }
        return true;
    }
    
    if (strcmp(message, "on") == 0) {
        if (battle_config.cashshop_redirect_to_adventure) {
            clif_displaymessage(fd, "Cash Shop Umleitung ist bereits aktiv.");
        } else {
            battle_config.cashshop_redirect_to_adventure = 1;
            clif_displaymessage(fd, "Cash Shop Button öffnet jetzt die Adventure Agency.");
            ShowInfo("Admin %s hat Cash Shop Umleitung aktiviert.\n", sd->status.name);
        }
    } else if (strcmp(message, "off") == 0) {
        if (!battle_config.cashshop_redirect_to_adventure) {
            clif_displaymessage(fd, "Cash Shop Umleitung ist bereits inaktiv.");
        } else {
            battle_config.cashshop_redirect_to_adventure = 0;
            clif_displaymessage(fd, "Normaler Cash Shop wird wieder geöffnet.");
            ShowInfo("Admin %s hat Cash Shop Umleitung deaktiviert.\n", sd->status.name);
        }
    }
    
    return true;
}
// ========== ✅ ENDE ==========
```

---

### 7.2: Broadcast an alle Spieler (Optional)

**📁 Datei: `src/map/atcommand.cpp`**

Wenn Sie möchten, dass alle Spieler benachrichtigt werden:

```cpp
ACMD(cashredirect) {
    // ... Code wie oben ...
    
    if (strcmp(message, "on") == 0 && !battle_config.cashshop_redirect_to_adventure) {
        battle_config.cashshop_redirect_to_adventure = 1;
        
        // ========== ✅ OPTIONAL: Broadcast hinzufügen ==========
        // Broadcast an alle Spieler
        char announce[128];
        sprintf(announce, "Der Cash Shop Button wurde zur Adventure Agency umgeleitet!");
        intif_broadcast(announce, strlen(announce) + 1, BC_DEFAULT);
        // ========== ✅ ENDE ==========
        
        ShowInfo("Admin %s hat Cash Shop Umleitung aktiviert.\n", sd->status.name);
    }
    
    // ... rest des Codes ...
}
```

---

## Kompilierung und Installation

### 1. Server neu kompilieren

```bash
cd /pfad/zu/rathena
./configure
make clean
make server
```

### 2. Server neustarten

```bash
# Alte Prozesse beenden
killall map-server

# Server neu starten
./map-server
```

### 3. Testen

Im Spiel als GM:

```
@cashredirect on      // Aktiviert Umleitung
@cashredirect off     // Deaktiviert Umleitung
@cashredirect status  // Zeigt aktuellen Status
```

---

## Fehlersuche

### Problem: Command nicht gefunden
- Prüfen Sie ob der Command in `atcommand_info[]` registriert wurde
- Neu kompilieren mit `make clean && make server`

### Problem: Cash Shop öffnet sich immer noch
- Prüfen Sie `battle_config.cashshop_redirect_to_adventure` Wert
- Debuggen Sie mit `ShowDebug` Ausgaben in `clif_parse_CashShopOpen`

### Problem: Adventure Agency öffnet nicht
- Prüfen Sie PACKETVER in `src/config/core.hpp`
- Adventure Agency benötigt PACKETVER >= 20111122
- Prüfen Sie ob Client die Adventure Agency unterstützt

### Debug-Code hinzufügen

```cpp
void clif_parse_CashShopOpen(int fd, struct map_session_data *sd) {
    ShowDebug("Cash Shop geöffnet, Redirect Status: %d\n", battle_config.cashshop_redirect_to_adventure);
    
    if (battle_config.cashshop_redirect_to_adventure) {
        ShowInfo("Leite zu Adventure Agency um für Spieler: %s\n", sd->status.name);
        clif_adventureagency_open(sd);
        return;
    }
    
    // ... rest ...
}
```

---

## 📦 ZUSAMMENFASSUNG - Alle hinzuzufügenden Code-Snippets

### 1️⃣ src/map/battle.hpp
```cpp
int cashshop_redirect_to_adventure;
```

### 2️⃣ src/map/battle.cpp (Initialisierung)
```cpp
0, // cashshop_redirect_to_adventure
```

### 3️⃣ src/map/battle.cpp (Config Array)
```cpp
{ "cashshop_redirect_to_adventure",     &battle_config.cashshop_redirect_to_adventure,      0,      0,      1       },
```

### 4️⃣ src/map/clif.cpp (in clif_parse_CashShopOpen nach nullpo_retv)
```cpp
// Prüfen ob Umleitung zur Adventure Agency aktiv ist
if (battle_config.cashshop_redirect_to_adventure) {
    // Öffne stattdessen Adventure Agency
    clif_adventureagency_open(sd);
    return;
}
```

### 5️⃣ src/map/clif.cpp (neue Funktion)
```cpp
void clif_adventureagency_open(struct map_session_data *sd) {
    int fd;
    nullpo_retv(sd);
    fd = sd->fd;
#if PACKETVER >= 20111122
    WFIFOHEAD(fd, packet_len(0x97f1));
    WFIFOW(fd, 0) = 0x97f1;
    WFIFOSET(fd, packet_len(0x97f1));
#else
    clif_messagecolor(&sd->bl, color_table[COLOR_RED], "Adventure Agency ist in diesem Client nicht verfügbar.", false, SELF);
#endif
}
```

### 6️⃣ src/map/clif.hpp
```cpp
void clif_adventureagency_open(struct map_session_data *sd);
```

### 7️⃣ src/map/atcommand.cpp (Funktion)
```cpp
ACMD(cashredirect) {
    if (battle_config.cashshop_redirect_to_adventure == 0) {
        battle_config.cashshop_redirect_to_adventure = 1;
        clif_displaymessage(fd, "Cash Shop Button öffnet jetzt die Adventure Agency.");
        ShowInfo("Admin %s hat Cash Shop Umleitung aktiviert.\n", sd->status.name);
    } else {
        battle_config.cashshop_redirect_to_adventure = 0;
        clif_displaymessage(fd, "Cash Shop Umleitung deaktiviert. Normaler Cash Shop wird geöffnet.");
        ShowInfo("Admin %s hat Cash Shop Umleitung deaktiviert.\n", sd->status.name);
    }
    return true;
}
```

### 8️⃣ src/map/atcommand.cpp (Registrierung)
```cpp
ACMD_DEF(cashredirect),
```

### 9️⃣ conf/battle/client.conf
```conf
//--------------------------------------------------------------
// Cash Shop Einstellungen
//--------------------------------------------------------------
cashshop_redirect_to_adventure: 0
```

---

## Wichtige Hinweise

1. **Backup erstellen**: Sichern Sie immer Ihre Dateien vor Änderungen
2. **Client Kompatibilität**: Stellen Sie sicher dass Ihr Client Adventure Agency unterstützt
3. **Packet Version**: Prüfen Sie PACKETVER Einstellungen
4. **Testing**: Testen Sie auf einem Testserver vor dem Live-Einsatz
5. **Logs**: Überwachen Sie die Server-Logs auf Fehler

---

## Lizenz und Credits

Diese Modifikation ist für rAthena Server gedacht und sollte mit der rAthena Lizenz kompatibel sein.
