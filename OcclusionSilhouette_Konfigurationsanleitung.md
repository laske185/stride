# OcclusionSilhouetteRenderFeature — Konfigurationsanleitung

## Was ist der Occlusion-Silhouette-Effekt?

Der Effekt zeigt eine farbige Silhouette von Objekten, wenn diese von anderer Geometrie (z.B. Gebaeude, Baeume) verdeckt werden. Typischer Einsatz: Spieleinheiten bleiben hinter Hindernissen sichtbar.

**Wichtig:** Silhouette-Objekte verdecken sich **nicht gegenseitig**. Nur Objekte ohne `OcclusionSilhouetteComponent` (z.B. Gebaeude) erzeugen Silhouetten. Das verhindert visuelles Chaos bei vielen Einheiten.

---

## Einrichtung im Graphics Compositor

### Schritt 1: Render Stage hinzufuegen

1. Oeffne den **Graphics Compositor** deines Projekts
2. Fuege einen neuen **Render Stage** hinzu
3. Konfiguriere:
   - **Name**: `OcclusionSilhouette` (frei waehlbar)
   - **Effect Slot Name**: `OcclusionSilhouette` (frei waehlbar)

### Schritt 2: Render Stage Selector hinzufuegen

1. Navigiere zum **Entry Points > Forward Renderer > Render Stage Selectors**
2. Fuege einen neuen **SimpleGroupToRenderStageSelector** hinzu
3. Konfiguriere:
   - **Render Stage**: den in Schritt 1 erstellten Stage (`OcclusionSilhouette`) auswaehlen
   - **Render Group**: `All` (damit alle Entities fuer den Stage in Frage kommen)
   - **Effect Name**: `OcclusionSilhouetteEffect`

### Schritt 3: OcclusionSilhouetteRenderFeature hinzufuegen

1. Navigiere zum **Mesh Render Feature > Render Features** im Graphics Compositor
2. Fuege ein neues **Occlusion Silhouette** Feature hinzu
3. Konfiguriere:
   - **Render Group**: `All` (Standard, kann eingeschraenkt werden)
   - **Opaque Render Stage**: den bestehenden `Opaque`-Stage auswaehlen
   - **Silhouette Render Stage**: den in Schritt 1 erstellten Stage auswaehlen

### Schritt 4: Silhouette Stage zum Renderer hinzufuegen

1. Navigiere zum **Entry Points > Forward Renderer > Renderers** (dort wo Opaque und Transparent gelistet sind)
2. Fuege einen neuen Renderer (z.B. **Single Stage Renderer** oder vergleichbar) fuer den Silhouette-Stage hinzu
3. Stelle sicher, dass der Silhouette-Stage **nach** dem Opaque-Stage gerendert wird (Reihenfolge ist wichtig, damit der Stencil-Buffer gefuellt ist)

---

## Einrichtung pro Entity

### Minimale Konfiguration

1. Waehle das Entity mit dem 3D-Modell aus
2. Klicke auf **Add Component**
3. Waehle **Rendering > Occlusion Silhouette**
4. Fertig — der Effekt nutzt die Standard-Farbe (halbtransparentes Cyan)

### Optionale Einstellungen

| Property | Standard | Beschreibung |
|---|---|---|
| **Enabled** | `true` | Silhouette ein-/ausschalten (auch zur Laufzeit) |
| **Silhouette Color** | Cyan (0, 0.75, 1, 0.6) | RGBA-Farbe der Silhouette. Alpha steuert die Transparenz. |

### Farb-Empfehlungen

| Einsatz | Farbe (RGBA) | Beschreibung |
|---|---|---|
| Eigene Einheiten | `(0, 0.75, 1, 0.6)` | Cyan, gut sichtbar |
| Feindliche Einheiten | `(1, 0.2, 0.2, 0.6)` | Rot, signalisiert Gefahr |
| Neutrale/NPCs | `(1, 0.9, 0.3, 0.5)` | Gelb, dezent |
| Hervorgehobene Objekte | `(0, 1, 0.5, 0.8)` | Gruen, hohe Sichtbarkeit |

---

## Laufzeit-Steuerung per Script

```csharp
// Silhouette ein-/ausschalten
var silhouette = entity.Get<OcclusionSilhouetteComponent>();
silhouette.Enabled = false; // Sofort unsichtbar

// Farbe aendern (z.B. bei Selektion)
silhouette.SilhouetteColor = new Color4(1f, 1f, 0f, 0.8f); // Gelb

// Komponente zur Laufzeit hinzufuegen
var comp = new OcclusionSilhouetteComponent
{
    SilhouetteColor = new Color4(1f, 0f, 0f, 0.6f)
};
entity.Add(comp);

// Komponente entfernen
entity.Remove<OcclusionSilhouetteComponent>();
```

---

## Typische Szenarien

### RTS-Spiel (z.B. "Diplomacy is not an Option")

- **Gebaeude, Baeume, Felsen**: KEIN `OcclusionSilhouetteComponent` — diese sind die "Waende"
- **Einheiten (eigene + feindliche)**: MIT `OcclusionSilhouetteComponent` — sichtbar hinter Hindernissen
- Ergebnis: Einheiten hinter Gebaeuden zeigen Silhouette, Einheiten hinter anderen Einheiten nicht

### Action-RPG (z.B. Diablo-Stil)

- **Umgebung (Mauern, Saeulen, Baeume)**: KEIN `OcclusionSilhouetteComponent`
- **Spieler-Charakter**: MIT `OcclusionSilhouetteComponent` (Cyan)
- **Gegner**: MIT `OcclusionSilhouetteComponent` (Rot) — optional nur bei Mouseover aktivieren

### Third-Person-Shooter

- **Level-Geometrie**: KEIN `OcclusionSilhouetteComponent`
- **Spieler-Charakter**: MIT `OcclusionSilhouetteComponent` — Charakter bleibt hinter Deckung sichtbar
- **Mitspieler**: MIT `OcclusionSilhouetteComponent` (andere Farbe)

---

## Einschraenkung per Render Group (optional)

Falls nicht alle Entities fuer den Silhouette-Effekt in Frage kommen sollen, kann die `RenderGroup` eingeschraenkt werden:

1. Im **OcclusionSilhouetteRenderFeature**: `Render Group` auf z.B. `Group1` setzen
2. Im **SimpleGroupToRenderStageSelector**: `Render Group` ebenfalls auf `Group1` setzen
3. Auf den Entities: `ModelComponent > Render Group` auf `Group1` setzen

Nur Entities in der passenden Gruppe werden dann verarbeitet.

---

## Fehlerbehebung

| Problem | Ursache | Loesung |
|---|---|---|
| Keine Silhouette sichtbar | Silhouette-Stage wird nicht nach Opaque gerendert | Render-Reihenfolge pruefen: Silhouette muss nach Opaque kommen |
| Keine Silhouette sichtbar | `OpaqueRenderStage` oder `SilhouetteRenderStage` nicht gesetzt | Beide Properties im RenderFeature konfigurieren |
| Alle Modelle schwarz | `RenderGroup = All` im Selector aber kein ColorWrite-Fix | Engine-Version aktualisieren (Fix ist enthalten) |
| Shader-Fehler E1202 | `.sdfx.cs`-Datei fehlt | Sicherstellen, dass `OcclusionSilhouetteEffect.sdfx.cs` vorhanden ist |
| NotImplementedException beim Laden | `[DataContract]` auf dem RenderFeature | RenderFeature darf kein eigenes `[DataContract]` haben |
| Silhouette an falscher Position (animiertes Modell) | Skinning nicht aktiv | Engine neu bauen — `.sdfx` inkludiert Skinning automatisch |
| Silhouette-Objekte verdecken sich gegenseitig | Wird nicht passieren | Alle Silhouette-Objekte teilen Stencil-Wert 1 — korrektes Verhalten |
