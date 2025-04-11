# SDUI Architecture with Jetpack Compose & ConstraintLayout

## 1. Why Server-Driven UI (SDUI)?
- SDUI will enable server-defined screens.
- Enahnce flixiblity and dynamic layout positioning.

---

## 2. Core JSON-Driven Approach
This design keeps **layout logic and view rendering entirely dictated by JSON**. Both wil be seperate from each other.

### JSON Segments per Component
Each view is represented as a flat JSON object with four primary sections:

```json
{
  "id": "view1",
  "type": "Text",
  "constraint": { ... },
  "ui": { ... }
}
```

Example json will be

```json
{
  "id": "ctaButton",
  "type": "Button",
  "ui": {
    "text": "Hello World",
    "fontSize": 16,
    "fontWeight": "bold",
    "fontColor": "#000000",
    "backgroundColor": "#FFFFFF",
    "cornerRadius": 8,
    "cornerType": "top",
    "borderColor": "#CCCCCC"
  },
  "constraints": {
    "start": "parent.start",
    "top": "previousView.bottom",
    "margin": {
    "start": 8,
    "top": 12
    },
    "width": "wrap",     // wrap | fill | fixed
    "height": "fixed",
    "heightValue": 48     // Only needed for fixed
  }
}
```

All Anchors are not mandatory, whatever is given will be included
Values: Another view's id.side or parent.side
Margins: Optional per side

#### Common (applies to most views)
- `padding` (dp)
- `backgroundColor` (hex)
- `borderColor` (hex)
- `cornerRadius` (dp)
- `cornerType`: `all`, `top`, `bottom`, etc.
- `width`, `height`: `wrap`, `fill`, `fixed` (with optional fixed size)

#### View-Specific
- **Text**: `text`, `fontSize`, `fontColor`, `fontWeight`
- **Button**: `text`, `actionId`, plus all common attributes
- **Image**: `src`, corner/border control
- **InputText**: `placeholder`, font styling, background, etc.

### Style Strategy
- Each component supports local styling via the `ui` key.
- Default style values are applied globally when missing.
- Shared style system is not needed — the system encourages localized, override-friendly customization.

### Flat Constraint Structure
- Uses `start`, `end`, `top`, `bottom`, margins.
- Anchors include IDs or keywords like `parent`, `titleText.start`.
- Width/height: `wrap`, `fill`, `fixed` with `widthDp`, `heightDp`.
- **Spacers are intentionally avoided** — spacing is handled using margins only for simplicity and fewer components in memory.

### Guidlines

Included as normal view in json, it will be like an invisible line helping to positioning of composables.

```json

{
  "id": "guideline_25",
  "type": "Guideline",
  "orientation": "vertical",
  "percent": 0.25    // according to screen size
}

```

---

## 3. Constraint Parsing & Layout Flow

### Parsing Pipeline
1. **Create references** for each component before building constraints.
2. **Collect anchors and margins** from JSON.
3. **Build `ConstraintSet`** using Compose DSL.

```kotlin
val constraintSet = ConstraintSet {
    val titleRef = createRefFor("titleText")
    val ctaRef = createRefFor("ctaButton")

    constrain(titleRef) {
        top.linkTo(parent.top, margin = 16.dp)
        start.linkTo(parent.start, margin = 24.dp)
    }

    constrain(ctaRef) {
        top.linkTo(titleRef.bottom, margin = 16.dp)
        start.linkTo(parent.start, margin = 24.dp)
        end.linkTo(parent.end, margin = 24.dp)
    }
}
```

### Layout Execution Strategy
- ConstraintSet creation and rendering must be sequential — views must not render before constraints exist.
- View references act as links; missing reference = invalid layout.

### ConstraintSet from JSON — Examples

#### Example 1: Text Start Aligned to Parent

```json
"constraints": {
  "start": "parent.start",
  "top": "parent.top",
  "marginStart": 16,
  "marginTop": 12
}

In Compose - 

```kotlin
constrain("titleText") {
    start.linkTo(parent.start, margin = 16.dp)
    top.linkTo(parent.top, margin = 12.dp)
}
```

Example 2: Image Next to Text with Guideline

```json
"constraints": {
  "start": "guideline1.start",
  "top": "parent.top",
  "end": "titleText.start",
  "marginEnd": 8
}
```

 In Compose:
 
 ```kotlin
 constrain("image1") {
    start.linkTo(guideline1.start)
    top.linkTo(parent.top)
    end.linkTo(titleText.start, margin = 8.dp)
}
```
---

## 4. Rendering Architecture

### Render Iterface
- A base interface with canHandle function to make sure correct data go in dispatch logic

```kotlin
interface ViewRenderer {
    fun canHandle(type: String): Boolean
    @Composable
    fun Render(viewId: String, json: JsonObject, refs: Map<String, ConstrainedLayoutReference>)
}
```

### Renderer Map
- Each view type is mapped using constant-time `associateBy`:
```kotlin
val rendererMap = listOf(
  TextRenderer(), ImageRenderer(), ButtonRenderer()
).associateBy { it.type() }
```
- Avoids conditionals or branching logic.
- No need to touch dispatch logic when adding new renderers.

### Renderer Flow Pseudocode
```kotlin
fun renderComponent(json: JsonObject): @Composable () -> Unit {
    val type = json["type"]?.asString ?: return MissingType()
    val renderer = rendererMap[type] ?: return UnknownType()
    return renderer.render(json)
}
```

### Text Renderer example

``kotlin
class TextRenderer : ViewRenderer {
    override fun canHandle(type: String) = type == "Text"

    @Composable
    override fun Render(id: String, json: JsonObject, refs: Map<String, ConstrainedLayoutReference>) {
        val ui = json["ui"].asJsonObject
        val text = ui["text"]?.asString ?: "⚠️ Missing Text"
        val modifier = buildModifier(ui) // common function fro all viewtypes to prepare modifier

        Text(
            text = text,
            modifier = modifier,
            fontSize = ui["fontSize"]?.asSp() ?: 14.sp,
            fontWeight = ui["fontWeight"]?.asFontWeight() ?: FontWeight.Normal,
            color = ui["fontColor"]?.toColor() ?: Color.Black
        )
    }
}
```

### Epoxy Integration (Temporary)
```kotlin
ViewType.VIEW_TYPE_COMPOSE_VIEW -> {
    widgetComposeView {
        id(index)
        onBind { _, view, _ ->
            view.findViewById<ComposeView>(R.id.compose_view).setContent {
                ComposeHelper.ComposeSectionBody(uiMetaInfo, childJson) {
                    triggerAction(it)
                }
            }
        }
    }
}
```

---

## 5. UI Defaults & Error Handling

### Defaults for non-essentials:
- `fontSize`: 14
- `backgroundColor`: `#FFFFFF`
- `cornerRadius`: 0
- `padding`: 0
- `borderWidth`: 0, `borderColor`: transparent

### Fallbacks:
- If essential key missing (e.g., `text` in `Text`): show placeholder text.
- If constraints missing: skip rendering that view.

---

## 6. Optimizations

- **Renderer Map:** Constant-time lookup via `associateBy`
- **Modifier reuse:** Reduces memory allocations
- **Avoid nesting:** Flat layout via `ConstraintLayout`
- **Memory control:** Use `remember` selectively
- **ConstraintSet:** Held per-screen, not global
- **Epoxy Compose injection:** Allows legacy integration during transition

---

## 7. Action Handling (Compose Lambda)
- Components declare `actionId`
- Passed via lambda to renderer
```kotlin
ComposeHelper.ComposeSectionBody(uiMetaInfo, jsonData) {
    performAction(actionId)
}
```
- Pure decoupled lambda; works across Epoxy or Compose natively

---

## 8. Logging System

### Structure
- Log unexpected states or missing essentials
- Hook into Firebase or internal collector
```kotlin
Log.e("SDUI", "Missing key: text in TextView")
```

---

## 9. Flow Diagram

### SDUI Component Flow
```
Download JSON → Store in DB → Read per screen →
Parse JSON → Build ConstraintSet → Compose View with Constraints →
Display in UI
```

---

## 10. Diagram: Parsing + Rendering
```
+----------------+       +-------------------+
| JSON per view  |  -->  | ConstraintParser  |  --+
+----------------+       +-------------------+    |
                                                  v
                                     +----------------------+     +------------------+
                                     | Renderer Dispatcher  | --> | Compose Function |
                                     +----------------------+     +------------------+
```

> ⚠️ Note: We mostly use `JsonObject` instead of typed data classes to avoid rigid models and maintain flexibility.
