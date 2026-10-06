# Issues

PrimeFaces Forum - Discussions about Ultima - #4875

- https://github.com/orgs/primefaces/discussions/4875



# Ultima template — anything to consider before migrating to PrimeFaces 16.0.0? #4875

Hi everyone,

I currently have a project using the Ultima template (version 7.0.0) with PrimeFaces, and I'm planning to migrate to PrimeFaces 16.0.0.

Just to clarify: my project is already 100% migrated to Jakarta (Jakarta EE namespaces, jakarta.* imports, etc.), so that part is already done and shouldn't be a blocker.

So far, the only change I made was updating the PrimeFaces version in my pom.xml. After that single change, I'm getting a compilation error. I'll attach a screenshot of the error below.

<img width="1531" height="244" alt="image" src="https://github.com/user-attachments/assets/110f4d28-b293-402c-bf83-b91c3d89a7e0" />

Before I dig further, I wanted to ask if there's anything specific about the Ultima theme/template I should take into account for this migration:

Do I need to update the designer/theme SASS files (theme-base folder) to match the new PrimeFaces version, as described in the general template migration guide?
Are there any breaking changes in 16.0.0 that specifically affect Ultima's layout, menu, or CSS structure?

Has anyone here already gone through this migration on a Jakarta-based project? Any tips, gotchas, or links to relevant migration notes would be very appreciated.

Thanks in advance!



# Running the Ultima layout/theme on PrimeFaces 16 — regressions & workarounds

**Setup**

- PrimeFaces **16.0.0** (Jakarta), PrimeFaces Extensions 16
- JSF 4.0 / Jakarta EE 11, WildFly, Java 21
- **Ultima** layout + theme package, originally downloaded for **PrimeFaces 15** (pre‑built  `primefaces-ultima-<color>[-dark]/theme.css` resource libraries + the `resources/layout/`  SCSS/JS + a custom `<pu:menu>` component extending `AbstractMenu`)

After bumping PrimeFaces 15 → 16 the app compiles/runs but there are several visual and build regressions, because PF16 moved structural CSS out of `components.css` into the theme and changed some component markup, while the Ultima assets are still the PF15 build.

Posting the findings + the workarounds we're using, in case it helps others in the same situation — and to ask whether an **Ultima release targeting PrimeFaces 16** is planned.

The good news first: **most of the Ultima theme survives**, because it uses descendant selectors (`body .ui-dialog .ui-dialog-content { … }`) that are immune to the new wrapper elements PF16 introduced. The breakage is a small, well‑defined set.

---

## 1. Client‑side widgets: `PrimeFaces.widget.X.extend({})` / `_super()` removed

**Symptom:** custom widgets and widget overrides throw `TypeError: …extend is not a function` at page load; the affected widgets never initialise.

**Cause:** PF16 widgets are ES6 classes; the old prototype‑emulation API is gone.

**Workaround:** convert to `class extends` + `super`:

```js
// before
PrimeFaces.widget.MyWidget = PrimeFaces.widget.BaseWidget.extend({
    init: function (cfg) { this._super(cfg); /* … */ }
});
// after
PrimeFaces.widget.MyWidget = class extends PrimeFaces.widget.BaseWidget {
    init(cfg) { super.init(cfg); /* … */ }
};
```

Overrides of existing widgets work the same way:

```js
PrimeFaces.widget.SelectOneMenu = class extends PrimeFaces.widget.SelectOneMenu {
    init(cfg) { super.init(cfg); /* … */ }
};
```

The Ultima `layout.js` also ships an override of `PrimeFaces.widget.Dialog` (the `enableModality` / `syncWindowResize` hack from issue [#924](https://github.com/primefaces/primefaces/issues/924), PF 5.3 era) — it can just be **deleted** on PF16.

---

## 2. Custom component extending `AbstractMenu` no longer compiles

**Symptom:**

```
X is not abstract and does not override abstract method getToggleEvent() in
org.primefaces.component.menu.AbstractMenu
```

**Cause:** PF16 promoted several properties to abstract on `AbstractMenu` and `RTLAware`.

**Workaround:** implement them (framework defaults shown):

```java
@Override public String  getToggleEvent() { return null; }
@Override public String  getTabindex()    { return "0"; }
@Override public boolean  isAutoDisplay()  { return true; }
@Override public int      getShowDelay()   { return 0; }
@Override public int      getHideDelay()   { return 0; }
@Override public String   getDir()         { return "ltr"; }   // from RTLAware
```

---

## 3. `p:dialog` — only the modal mask shows; dialog is invisible / off to the left

**Symptom:** opening any `p:dialog` dims the screen but the dialog box is not visible, or
appears far to the left instead of centered. Affects **all** dialogs.

**Cause:** PF16 split the dialog markup in two:

```xml
<div class="ui-dialog …">              <!-- now a full-viewport scroll layer -->
  <div class="ui-dialog-box ui-shadow"> <!-- the actual visible box (NEW) -->
    <div class="ui-dialog-titlebar">…</div>
    <div class="ui-dialog-content">…</div>
    <div class="ui-dialog-footer">…</div>
  </div>
</div>
```

- `.ui-dialog` used to be the box; it is now the scrolling layer and must be   `position:fixed; inset:0; overflow-y:auto`. A PF15 theme still styles it as the box  (shadow/radius/padding), so it never becomes a layer.
- `.ui-dialog-box` is brand new and a PF15 theme has **no rules** for it.
- The widget centers the box *inside* `.ui-dialog` (`box.position({ of: this.jq })`), so  if `.ui-dialog` isn't the full viewport the centering math is off.
- `DialogRenderer` now writes the component's **`style=`** onto `.ui-dialog` (the layer),  not the box. Pages that size the dialog via `style="width: …"` instead of the  **`width=`** attribute end up with a narrow, left‑anchored layer → box centered off to
  the left. `initSize()` applies `width` from the **attribute** to `.ui-dialog-box`, so  the attribute is now the only correct place for sizing.

**Workaround** (port the structural bits from a PF16 theme, e.g. `saga`):

```scss
body .ui-dialog {
    position: fixed;
    top: 0; right: 0; bottom: 0; left: 0;
    margin-left: auto; margin-right: auto;   /* re-center layer if a page put width/max-width in style= */
    box-sizing: border-box;
    padding: 1.5rem;
    overflow-x: hidden;
    overflow-y: auto;
}
body .ui-dialog.ui-overflow-hidden,
body .ui-dialog.ui-dialog-absolute,
body .ui-dialog.ui-dialog-fitviewport { overflow-y: hidden; }

body .ui-dialog .ui-dialog-box {
    position: relative;
    display: inline-flex;
    flex-direction: column;
    border-radius: 4px;
    box-shadow: 0 11px 15px -7px rgba(0,0,0,.2),
                0 24px 38px 3px rgba(0,0,0,.14),
                0 9px 46px 8px rgba(0,0,0,.12);
    background: var(--surface-e, #fff);
}
body .ui-dialog.ui-dialog-absolute .ui-dialog-box { position: absolute; }
body.ui-dialog-open { position: relative; }
```

Plus, in the views: move `width` / `max-width` off `style=` — use the `width=` attribute (and target `.ui-dialog-box` for `max-width` in a page `<style>` if needed).

The title bar / content / footer background & colours do **not** need patching — the PF15 theme's `body .ui-dialog .ui-dialog-content { … }` etc. still match (descendant selector) and win on specificity.

---

## 4. Bullet markers on `p:menu`, `p:menuButton`, `p:tieredMenu`, `p:contextMenu`, `p:slideMenu`, `p:panelMenu`

**Symptom:** the overlay `<ul>` of these components shows default list bullets and the
default `<ul>` indentation.

**Cause:** PF16 `MenuRenderer` no longer adds `ui-helper-reset` to `<ul class="ui-menu-list">` (it writes just `Menu.LIST_CLASS = "ui-menu-list"`). PF15 themes relied on `ui-helper-reset` for the `list-style:none` reset. A PF16 theme resets it directly on `.ui-menu-list`.

**Workaround:**

```scss
body .ui-menu .ui-menu-list,
body .ui-panelmenu .ui-menu-list {
    list-style: none;
    margin: 0;
    padding: 0;
}
```

---

## 5. `p:fileUpload` (drag & drop): a permanent overlay covers the component

**Symptom:** the advanced `p:fileUpload` shows a full‑size overlay (icon + area) on top of itself at all times.

**Cause:** PF16 added `<div class="ui-fileupload-drag-overlay">` **without an inline style**. The theme is expected to hide it (`display:none`) and reveal it only while a file is dragged over (`.ui-state-drag`). PF15 themes have no rule for it.

**Workaround:**

```scss
body .ui-fileupload .ui-fileupload-drag-overlay {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    z-index: 1;
    display: none;
    align-items: center;
    justify-content: center;
}
body .ui-fileupload.ui-state-drag .ui-fileupload-drag-overlay { display: flex; }
```

---

## 6. Smaller API / attribute changes surfaced during the migration

| Component | Change |
|---|---|
| `p:confirmPopup` | `showEffect` / `hideEffect` attributes removed |
| `p:accordionPanel` | `activeIndex` → `active` (`activeIndex` deprecated). Both are `String`; an `int` bean property still works via EL coercion |
| `p:graphicImage` | `cache` default `true` → `false` (only affects `value=` with a `String`/`StreamedContent`; a cache‑buster `uid` is appended per render) |
| `p:separator` | removed → `p:divider` |
| DataTable | CSS class `ui-filter-togglable` → `ui-filter-toggleable` |
| `p:selectOneButton` | option label markup `span.ui-button-text` → `label.ui-button-text` (breaks custom CSS/JS selectors) |

---

## PrimeFlex note

PrimeFlex 4 renamed the colour CSS variables its utility classes consume (`--blue-500` → `--p-blue-500`, `--surface-0`, …). A PF15 Ultima theme provides the un‑prefixed names, so PrimeFlex 4's colour/surface utilities resolve to nothing. Staying on **PrimeFlex 3** (`primeflex.min.css`) works with the PF15 theme tokens; PrimeFlex 4 needs either the `primeflex/themes/primeone-*.css` token file or a regenerated PF16 theme.

---

## Question

I noticed that the latest available Ultima version is currently 15.0.1, which appears to target PrimeFaces 15.

Is there a new Ultima release specifically targeting **PrimeFaces 16** planned?

Thanks!
