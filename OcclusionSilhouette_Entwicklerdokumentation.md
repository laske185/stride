# OcclusionSilhouetteRenderFeature — Entwicklerdokumentation

## Zusammenfassung

Das **OcclusionSilhouetteRenderFeature** rendert eine halbtransparente, einfarbige Silhouette von Meshes, wenn diese von anderer Geometrie verdeckt werden (X-Ray / See-Through-Effekt). Der Effekt wird typischerweise eingesetzt, um Spieleinheiten hinter Gebaeuden oder Baeumen sichtbar zu machen.

---

## Architektur

### Projektaufteilung

Die Implementierung ist auf zwei Projekte verteilt, da `Stride.Rendering` keine Referenz auf `Stride.Engine` hat (die Abhaengigkeit geht umgekehrt):

| Datei | Projekt | Zweck |
|---|---|---|
| `OcclusionSilhouetteComponent.cs` | Stride.Engine | Daten-Komponente fuer Entities |
| `OcclusionSilhouetteRenderFeature.cs` | Stride.Engine | SubRenderFeature mit Rendering-Logik |
| `OcclusionSilhouetteShader.sdsl` | Stride.Rendering | HLSL-Shader (Flat-Color-Ausgabe) |
| `OcclusionSilhouetteEffect.sdfx` | Stride.Rendering | Effekt-Komposition (Transformation + Skinning) |
| `OcclusionSilhouetteEffect.sdfx.cs` | Stride.Rendering | Registrierung des Effekts beim ShaderMixinManager |
| `OcclusionSilhouetteShaderKeys.cs` | Stride.Rendering | Parameter-Keys fuer Shader-Zugriff |

### Dateipfade

```
sources/engine/Stride.Engine/
  Engine/
    OcclusionSilhouetteComponent.cs
  Rendering/OcclusionSilhouette/
    OcclusionSilhouetteRenderFeature.cs

sources/engine/Stride.Rendering/
  Rendering/OcclusionSilhouette/
    OcclusionSilhouetteShader.sdsl
    OcclusionSilhouetteEffect.sdfx
    OcclusionSilhouetteEffect.sdfx.cs
    OcclusionSilhouetteShaderKeys.cs
```

---

## Funktionsweise

### Zwei-Pass-Rendering mit Stencil-Buffer

Der Effekt nutzt zwei Render-Stages und den Stencil-Buffer, um Self-Occlusion zu vermeiden:

**Pass 1 — Opaque Stage (normales Rendering):**
- Alle Entities mit `OcclusionSilhouetteComponent` schreiben Stencil-Wert `1` (Bit 0) wo ihre Pixel gerendert werden
- Das normale Material-Rendering bleibt unveraendert
- Der Depth-Buffer wird normal gefuellt

**Pass 2 — Silhouette Stage:**
- Depth-Test: `CompareFunction.Greater` — nur dort zeichnen, wo das Objekt HINTER anderer Geometrie liegt
- Stencil-Test: `CompareFunction.NotEqual` (Referenz 1) — Pixel ablehnen, die von Silhouette-Objekten stammen
- Depth-Write: deaktiviert — Depth-Buffer nicht verfaelschen
- Blend-State: `AlphaBlend` — halbtransparente Ueberblendung
- Entities OHNE Komponente: `ColorWriteChannels.None` — keine schwarze Ueberzeichnung

### Self-Occlusion-Vermeidung

Alle Silhouette-Objekte teilen denselben Stencil-Wert (1). Dadurch:
- Ein Silhouette-Objekt erzeugt **keine** Silhouette fuer ein anderes Silhouette-Objekt
- Silhouetten erscheinen nur hinter "echten" Hindernissen (Objekte ohne Komponente)
- Beispiel: Bei einem RTS mit vielen Einheiten hinter Gebaeuden sieht man Silhouetten nur hinter Gebaeuden, nicht hinter anderen Einheiten

### Skinning-Unterstuetzung (animierte Objekte)

Die `.sdfx`-Datei steuert die Shader-Komposition. Sie prueft zur Compile-Zeit die Permutation `MaterialKeys.HasSkinningPosition` und fuegt bei Bedarf den `TransformationSkinning`-Mixin hinzu. Das funktioniert, weil:

1. `SkinningRenderFeature.PrepareEffectPermutations()` validiert Skinning-Parameter auf **allen** Effect-Slots (alle Render-Stages)
2. Die `.sdfx` reagiert auf diese Permutationen und inkludiert `TransformationSkinning`
3. `SkinningRenderFeature.Prepare()` laedt die `BlendMatrixArray` in den Constant-Buffer

Dadurch nutzt der Silhouette-Pass automatisch die korrekte animierte Pose.

### .sdfx.cs — Warum diese Datei noetig ist

Jede `.sdfx`-Datei benoetigt eine begleitende `.sdfx.cs`-Datei. Diese wird normalerweise vom Stride Visual Studio Package generiert und muss ins Repository eingecheckt werden. Sie enthaelt:
- Eine `IShaderMixinBuilder`-Implementierung (C#-Uebersetzung der .sdfx-Logik)
- Einen `[ModuleInitializer]`, der den Effekt beim `ShaderMixinManager` registriert

Ohne diese Datei findet der Shader-Compiler den Effekt nicht und meldet: `E1202: The mixin [OcclusionSilhouetteEffect] dependency is not in the module`.

---

## Komponenten im Detail

### OcclusionSilhouetteComponent

```csharp
[DataContract("OcclusionSilhouetteComponent")]
[Display("Occlusion Silhouette", Expand = ExpandRule.Once)]
[ComponentCategory("Rendering")]
public sealed class OcclusionSilhouetteComponent : ActivableEntityComponent
{
    [DataMember(10)]
    public Color4 SilhouetteColor { get; set; } = new Color4(0f, 0.75f, 1f, 0.6f);
}
```

- Erbt von `ActivableEntityComponent` — stellt `Enabled`-Property bereit
- Reiner Daten-Container, keine Logik
- `SilhouetteColor`: RGBA-Farbe (Standard: halbtransparentes Cyan)

### OcclusionSilhouetteRenderFeature

Erbt von `SubRenderFeature` und wird in die `MeshRenderFeature.RenderFeatures`-Liste eingehaengt.

**Kein eigenes `[DataContract]`!** Die Basisklasse `RenderFeature` hat `[DataContract(Inherited = true, DefaultMemberMode = DataMemberMode.Never)]`. Ein zusaetzliches `[DataContract]` wuerde den `DefaultMemberMode` zuruecksetzen und eine `NotImplementedException` beim Laden ausloesen.

**Lifecycle-Methoden:**

| Methode | Phase | Aufgabe |
|---|---|---|
| `InitializeCore()` | Einmalig | RenderData-Key und CBuffer-Slot anlegen |
| `Extract()` | Pro Frame | Komponenten-Daten aus Entities extrahieren und in RenderData speichern |
| `Prepare()` | Pro Frame | Farb-Daten in den GPU Constant-Buffer schreiben |
| `Draw()` | Pro Stage-Batch | Stencil-Referenz setzen (vor MeshRenderFeature-Draw) |
| `ProcessPipelineState()` | Pipeline-Kompilierung | Depth/Stencil/Blend-State je nach Stage konfigurieren |

**Datenfluss:**

```
Entity (OcclusionSilhouetteComponent)
  |
  v  [Extract]
ModelComponent.Entity.Get<OcclusionSilhouetteComponent>()
  |
  v  color -> RenderData[ObjectNodeReference]
  |
  v  [Prepare]
color -> ConstantBuffer (PerDraw.SilhouetteColor)
  |
  v  [ProcessPipelineState]
PipelineState konfiguriert (Depth/Stencil/Blend)
  |
  v  [MeshRenderFeature.Draw]
GPU rendert mit konfiguriertem State + Shader
```

### OcclusionSilhouetteShader.sdsl

```hlsl
shader OcclusionSilhouetteShader : ShaderBase, Global
{
    cbuffer PerDraw
    {
        [Color]
        stage float4 SilhouetteColor;
    }

    stage override void PSMain()
    {
        float pulse = 0.85 + 0.15 * sin(Global.Time * 3.0);
        streams.ColorTarget = float4(SilhouetteColor.rgb * pulse, SilhouetteColor.a);
    }
};
```

- Erbt von `ShaderBase` (stellt `PSMain`, `streams.ColorTarget` bereit) und `Global` (stellt `Global.Time` bereit)
- Gibt eine Flat-Color aus — kein Lighting, kein Texture-Sampling
- Subtiler Puls-Effekt ueber Sinus-Funktion (85% bis 100% Helligkeit, 3 Hz)

### OcclusionSilhouetteEffect.sdfx

Effekt-Komposition die zur Compile-Zeit entscheidet, welche Mixins inkludiert werden:

```
ShaderBase + TransformationBase + TransformationWAndVP
  + (optional) TransformationSkinning       // bei Skelett-Animation
  + (optional) TransformationWAndVPInstanced // bei GPU-Instancing
  + OcclusionSilhouetteShader               // Pixel-Ausgabe
```

---

## Stencil-Bit-Nutzung

Das Feature nutzt **nur Bit 0** des Stencil-Buffers (Masken `0x01`). Die Bits 1-7 bleiben frei fuer andere Features. Falls ein anderes Feature ebenfalls Bit 0 nutzt, muss die Maske angepasst werden.

---

## Bekannte Einschraenkungen

- **Transparente Objekte**: Objekte mit transparentem Material im Transparent-Stage werden vom Stencil-Marking im Opaque-Stage nicht erfasst. Sie erzeugen keine Silhouetten-Blockierung.
- **MSAA**: Keine spezielle MSAA-Behandlung. Der Effekt funktioniert, aber Kanten koennen Aliasing zeigen.
- **Mehrere Silhouette-Farben hinter demselben Hindernis**: Ueberlappende Silhouetten verschiedener Entities blenden sich additiv (Alpha-Blend).
