# FEP-0015 Icon Theming

| FEP-0015       |                                                                                                 |
| ------- | ------------------------------------------------ |
| Type           | Core Change                                                                                     |
| Status         | Draft                                                                                           |
| Author(s)      | Kacper Donat @kadet1090                                                                         |
| Version        | 0.3                                                                                             |
| Created        | 2026-09-09                                                                                      |
| Updated        | 2026-09-09                                                                                      |
| Discussion     | n/a                                                                                             |
| Implementation | n/a                                                                                             |

Defines a new solution for icon themes and recolored pixmaps in FreeCAD.

## Motivation

FreeCAD resolves icons by name today, but nothing about that resolution is configurable.
`BitmapFactoryInst::pixmap()` walks a fixed sequence - an absolute path, then the `icons:` Qt
search path with a fixed list of extensions, then an optional external theme directory - and ends
at a hardcoded placeholder. Which file a name reaches is therefore a property of the compiled binary
and of whichever directories happen to be registered, not something a theme can state. While several
hacky solutions like addition of custom Qt Resources to replace some icons and alter search path
exists, none of them is fully supported in core.

The icon support in FreeCAD in general has few important drawbacks:
- **An icon set cannot be easily swapped.** Shipping a second look for FreeCAD means replacing files
  in place or using custom qt resources - which does not always work reliably.
- **Icons are tightly coupled with filename.** Icon names are based on files and uses are not always
  semantic - a different features can use icon from outside of its domain because it looks good,
  even if semantically it does not make sense.
- **Icon appearance cannot follow the interface theme.** It's not possible to use icons based on
  stroke as it's not possible to recolor the icons for specific use. Theme developers need to
  provide variants of icons for dark and light modes now. Icons cannot also include accent colors.
- **Stroke weight cannot follow icon size.** Same goes for the icon stroke - stroke thickness at
  different sizes is perceived visually in a different way. 2px stroke at 64px feels thin, but 2px
  stroke at 16px is huge.

There are also other defects that follow from the same fixed pipeline. Every SVG is rendered at a
hardcoded 64x64 regardless of the size that will be drawn - while it's good enough normally because
we don't use larger icons it is still a design flaw.

## Rationale

This FEP proposes making FreeCAD's icon set a themeable declarative resource. A declarative icon
theme file would state how an icon name maps to a file and how that file is processed before it is
drawn, so that an icon set could be replaced, recolored or restroked without recompiling and without
editing the several hundred call sites that ask for icons by name.

Icon lookup is currently compiled in: `BitmapFactoryInst::pixmap()` walks a fixed sequence of
extensions over the `icons:` search path and ends at a hardcoded placeholder. Anything a theme needs
to change about that has to be expressible in a file it ships, since theme authors do not rebuild
FreeCAD. The freedesktop icon theme specification takes the same approach with `index.theme`.

Names are resolved by ordered rewrite rules rather than by a name-to-file table or a list of search
directories. A table is impractical: FreeCAD has several thousand icon names, and enumerating them
would fix in place the accidental sharing described above rather than give a theme a way to undo it.
A directory list is what exists today and cannot rename anything - each directory registered through
`addPath()`, of which Part alone registers four, is searched for every icon name in the application
rather than for the icons it contains. Rewrite rules express a family redirect, a catch-all and a
single remapping with one construct, and decouple the requested name from the answering file.

A rule applies only when its search paths resolve to a file that exists. Icon sets are typically
partial, and on a name match alone a rule matching `PartDesign_(.+)` would blank every PartDesign
icon the theme had not drawn. The existence condition bounds a rule to what it has artwork for and
lets the remainder fall through, which is also what makes a workbench-scoped rule safe to leave
registered permanently. This also allows for graceful degradation - if some new icon is not yet
created in the set we are currently using - it will fallback to the default rule and not apply
potentially wrong processing rules.

The suggested solution also allows theme authors to apply specific overrides to icons and remap them
to something else. For example author can define one file that defines a representation of a box.
With palette change that icon can be used both for Surface workbench (Surface_Box - with yellow
color), Part (Part_Box - with blue color) and as generic object icon (leaving it gray).

Rules arrive from the icon theme, (including the files it inherits) and from any workbench that
registers one, so consultation order cannot be load order - module load order varies with the
installed addons. Modules are expected to provide defaults to serve as fallback for icon resolution
- just like it is now. Explicit priorities, and a tier holding contributed rules below everything
the theme declares, make the winner a function of what was written rather than of when it was read.

### Icon processing

A resolved file is treated as a source for the resulting pixmap, but the resulting pixmap does not
need to match the file exactly. Themes can specific simple processing rules to alter looks of one
file to for example swap color of lines for light or dark theme, change accent color or alter stroke
width.

A color can be supplied to replace the `currentColor`, an SVG keyword under which an element takes
the color its context gives it - usually the color used for the text. This is primarily intended to
replace a need for dedicated light and dark icons. It can be used also for other things however -
for actions that are considered dangerous UI may set that color to be `@DangerColor` (usually red).
This is a value that will be controlled mostly from outside of the icon set - within a style that is
used to render UI widgets.

Other colors in the icon (for example faces of a box in part design primitive) may be replaced using
palette swap mechanic - every occurrence of specified color in the palette swap will be replaced
with color specified in theme. That color may come from UI theme for example so the icon set may
react to changes in the UI look and cooperate with UI theme to achieve desired looks.

Other styling can be achieved with CSS features. Classes can be added to a document, and a
stylesheet injected into it. Together these cover what applies to some elements and not others: a
size that wants a lighter stroke declares `path, line { stroke-width: 1.5px; }`, and a variant that
should read as muted adds a class the stylesheet answers. Example of such modification is change to
a stroke weight - 2px reads as thin at 64px and heavy at 16px, so no single drawing is correct at
every size.

Classes can be also used to override other aspects of the SVG icon, change stroke style or even show
or hide specific layers based on the requirements. This allows designers to create one master file
and then use icon theming mechanics to alter the looks to match desired state.

KDE has shipped something close for years, and it is the nearest precedent for this half of the
proposal. A Breeze icon carries a `<style>` element under a known id declaring classes such as
`.ColorScheme-Text` and `.ColorScheme-Accent`; elements reference those classes alongside
`fill="currentColor"`; the theme opts in through a `FollowsColorScheme` key in its `index.theme`;
and at load `KIconLoader` replaces that element's contents with a stylesheet generated from eight
named color roles and the icon's state. The difference is where the requirement falls. Breeze
rewrites a stylesheet the artwork has to contain already, so an icon drawn without one cannot be
themed - roughly two thirds of Breeze's own are in that position. Injecting a stylesheet asks
nothing of the artwork.

### Variants and overrides

Some features require icons with subtle variations that may be hard to justify using separate icons
for it, especially when there are multiple different attributes that can be altered. One example is
icons used for sketcher elements panel. Now the icons are recolored by the workbench itself in
`ElementWidgetIcons::getMultIcon()` implementing that logic by hand. Instead of doing it by hand
every time such feature is needed the proposal suggests to split the responsibilities - sketcher
should only put a request for an icon with specifics features and it should be a job of an icon
manager to deliver correct icon.

Given that multiple variants of an icon may be active at the same time, variants can only alter the
processing of the icon - usually adding CSS class to the SVG file to alter the looks in some way.

Since injected CSS and other processing attributes can reference style parameters, the code can also supply direct overrides for style parameters when requesting icons. Such parameters can be used to control certain aspects of the icon. One example of case that couldn't be handle with enumerated variants is passing a color of the body to the body icon - so the icon matches actual rendered geometry. The proposal does not suggest that this is the right thing to do from UX perspective, but it is possible with the proposal.

### Re-rendering icons at runtime

Recoloring one rendering requires recovering the source file from an already drawn pixmap. Qt's
styling entry points hold a `QPixmap` without the `QIcon` behind it - `QStyle::generatedPixmap()`
receives a pixmap and returns a variant of it - because Qt assumes variants are produced by
filtering, a blue tint for selected or desaturation for disabled. Filtering suits bitmaps and not
stroke artwork, where the required result is the same drawing rendered again in another color.

Which color a widget's icon takes is left to the style system. An icon theme cannot know that an
icon sits on a primary button or a hovered row, and answering it here would either duplicate the
token machinery or constrain it. This FEP proposes the mechanism; the policy is expected in a
separate FEP.

The icon theme carries its own preference rather than belonging to the interface theme because the
two vary independently: an icon set is wanted under whichever interface theme runs, and an interface
theme need not ship one, though it may suggest one.

Designs set aside - icon rules inside the interface theme file, inferring a name from a path by its
shape, and resolving the theme's expressions once at load time - are discussed under [Rejected
Ideas](#rejected-ideas) and [Alternatives](#alternatives).

## Specification

Three things are specified: the manifest that describes an icon set, `Gui::IconManager`, which reads
it and answers requests for icons, and `Gui::ParametrizedIconEngine`, the `QIconEngine` that renders
through the manager so an icon reflects the manifest each time it is drawn rather than at the moment
it was created. The remainder of this section describes them in the present tense, as a
specification of what would hold were the proposal adopted.

### Usage

Nothing is required of code that asks for an icon today. Both entry points keep working and gain
theme resolution and processing without being edited:

```cpp
QPixmap pixmap = Gui::BitmapFactory().pixmap("PartDesign_Body");
```

Code that wants a live icon asks the manager for one. The request is defaulted throughout, so the
common case carries nothing:

```cpp
button->setIcon(IconManager::instance().icon("Std_Refresh"));
button->setIcon(IconManager::instance().icon(
    "Std_Refresh",
    RenderRequest {}.withColor(palette.buttonText().color())
));
```

A call-site that has additional information to be passed to the manager can do that with fluent
interface of the RenderRequest. The information may be for example a variant of icon (like normal vs
construction geometry for sketcher), color override or parameters override. It's not possible to
define specific processing rules here - the call must be agnostic and processing is highly dependent
on the file itself.

```cpp
menu->addAction(
    IconManager::instance().icon("Std_Delete", RenderRequest {}.withVariants({"danger"})),
    tr("Delete")
);

const QPixmap pixmap = IconManager::instance().pixmap(
    "Tree_Item",
    RenderRequest {}
        .withSize(16) // shorthand for .withSize({ 16, 16 })
        .withColor()
        .withParameters({{"CurrentColor", Base::Color::fromValue<QColor>(item->accent())}})
);
```

From Python the same three things read as keyword arguments, with values written the way Python
writes them:

```python
Gui.icon("Std_Delete", variants=["danger"])
Gui.pixmap("Tree_Item", size=16, parameters={"CurrentColor": "#e01b24"})
```

A workbench describing where its own icons live adds one line beside the `addIconPath` it already
calls:

```python
FreeCADGui.addIconRules(":/PartDesign/icons.yaml")
```

### The icon theme file

An icon theme is a YAML file at `qss:icons/{IconTheme}.yaml`, where `IconTheme` comes from the
`BaseApp/Preferences/Bitmaps/Theme/IconTheme` preference. Icon themes can be base on other themes
using the `_inherits` mechanism that merges multiple files together.

```yaml
_inherits: [defaults.yaml]

defaults:
  searchPaths:
    - ":/icons/{}.svg"

  processing:
    svg:
      currentColor: "@BaseTextColor"
      addCss: |
        path, line, circle { stroke-width: 2px; }
        .muted { opacity: 0.5; }

  sizes:
    16px:
      searchPaths:
        - ":/icons/16/{}.svg" # look for dedicated icon for the size first
        - ":/icons/{}.svg"
      processing:
        svg:
          addCss: |
            path, line, circle { stroke-width: 1.5px; }

  variants:
    danger:
      processing:
        svg:
          currentColor: "@Red.500"
    muted:
      processing:
        svg:
          addCssClasses: ['muted']

rules:
  - match: "PartDesign_(.+)"
    priority: 10
    searchPaths:
      - ":/icons/pd/$1.svg"
    processing:
      svg:
        paletteSwap:
          "#000000": "@AccentColor"

icons:
  PartDesign_Body: ":/icons/pd/body-filled.svg"
  PartDesign_Pad: Part_Extrude
  Std_Refresh:
    searchPaths: [":/icons/tabler/outline/refresh.svg"]
    processing:
      svg:
        currentColor: "reset()"

missing: "help-browser"
```

A rule states which icon names it applies to, through an regular expression in specified `match`,
and defines `searchPaths` where the icon can possibly be found. As noted earlier, the search paths
are looked through in definition order. The `processing` section defines rules for processing the
icon. The solution allows overrides of the rules for specific `sizes` - for example a different
processing or even a different search paths can be applied. Each rule also defines a priority - so
when merging multiple files it is possible to still get proper rule order.

A `variants` block is shaped like `sizes` and works the same way, except that the variants must be
specified explicitly on the call-site and don't always exist. Because multiple variants can be
active at once, a variant supplies processing only and never contributes search paths, i.e. it's
not possible to override file for a specific variant due to possible conflicts.

The `searchPaths` holds path templates. `{}` stands for the requested name and `$0` for the whole
match, while `$1`…`$n` are the capture groups of the enclosing rule's `match`; inside `defaults`,
which has no pattern, only `{}` and `$0` are available.

`priority` is an integer defaulting to 0, and rules are considered from the highest downwards. Ties
are broken toward the more derived file, so a file's own rules are considered before those it
inherits, and then by declaration order within a file.  The field exists so that rules gathered from
several files fall into a deliberate order rather than the order they happened to be loaded in.

Processing is keyed by source format. For now only `svg` is defined - but other processing options
can be introduced later if needed without need for separate FEP - so files in other formats resolve
and rasterize untouched. The key exists so that a future raster-processing block does not reshape
the file.

Every value in processing holds a style parameter expression rather than hardcoded value.
Expressions are evaluated at render (pixmap creation - not paint) time rather than at load time, so
an interface theme change recolors icons without the icon theme being reloaded or reparsed.

#### Baseline aka `defaults`

The `defaults` carries the general fallback if no rule is matched, and determines the baseline. If
some processing is defined in the defaults section - it will be applied for every icon, regardless
of the rule matched if not explicitly opted out by the rule.

#### Specific icon overrides

`icons` is the way to remap icons or provide specific overrides form them. It can be used to provide
a concrete mapping between semantic icon name and a file, or provide specific overrides for specific
icon A value written as a plain string is taken as a file if a file is there, and as the name of
another icon if not. `PartDesign_Body` above names a file; `PartDesign_Pad` names `Part_Extrude`,
which is then resolved exactly as though `Part_Extrude` had been asked for in the first place -
through its own override, the rules, and `defaults`. The two readings do not have to be told apart
by how the string looks, because the manifest already answers every question of this kind the same
way: try it, and if it is not there, carry on to the next possibility.

The longer form spells out what a bare string leaves implicit, and is what to use when the override
carries more than a destination:

```yaml
icons:
  Part_Cut:
    alias: Part_Boolean
    processing:
      svg:
        currentColor: "reset()"
```

An alias is followed from the top for the name it names, so it reaches whatever that name would have
reached, in any manifest. A chain that returns to a name already visited is abandoned with one
warning, and the original name carries on through the rules as though the override had not matched.

#### Others

The `missing:` key names the icon drawn when nothing resolves, replacing the hardcoded
`help-browser`.

### How a rule is chosen

A rule applies to a request when its `match` accepts the name *and* one of its search paths names a
file that exists. Both conditions are required: a rule that claims a family of names but ships no
artwork for the particular one being asked about does not take effect, and the next rule down is
tried instead. A theme may therefore override as much or as little as it has files for. There is no
risk of one rule affecting files from another - it is perfectly valid to have two rules that both
match for `PartDesign_(.+)` and refer to filled or outline icons - with different processing rules.

Within one manifest, the `icons` map is consulted first, then the rules in priority order, then
`defaults`. The first entry that satisfies both conditions is the applied one. It supplies the file,
and it is also the entry whose `processing` is used - whatever found the icon is what styles it.

An `icons` entry is subject to the same two conditions as a rule. An override whose file is not
there is tried as an alias, and one that is neither a file nor a name anything answers to falls
through to the rules rather than leaving the icon blank. Since such an entry can only be a mistake
in the manifest, that is reported the first time the name is resolved.

When an alias supplies the icon, the entry that won for the aliased name supplies the processing
too, and the aliasing override is consulted ahead of it - it is the more specific statement about
the icon that was actually asked for. `Part_Cut` above therefore takes `Part_Boolean`'s file and its
processing, less the recoloring it turns off for itself.

A rule that declares no `searchPaths` can never satisfy the second condition. That is a theme
authoring mistake rather than a way to style a family in place, and it is reported once when the
manifest is loaded.

Within the applied rule, the blocks of the request's active variants are consulted first, then its
size bucket, then the rule itself. A variant outranks a size bucket because it is something the call
site asked for where a size is ambient. Where several variants are active and more than one declares
the same key, they are consulted in the order the manifest declares them, so the outcome depends on
the manifest rather than on how a caller happened to order its list.

The size bucket may override search paths - which is how a set that ships a separate drawing at
16px offers it. Size buckets are chosen by taking the nearest declared size at or above the
requested one, floored at the smallest declared bucket, so a layer declares a bucket only where
something genuinely differs. If the size requested is larger than largest bucket - defaults are
used. Reasoning is that it's always better to downscale icons rather than upscale, and small icons
are ones which can require dedicated icons.

`processing` then resolves one key at a time: the applied rule's active variants, its size bucket
and then the rule itself, followed by `defaults`' active variants, its size bucket and `defaults`
itself, with the first of those to declare a key supplying it. In the manifest above,
`PartDesign_Body` rendered at 16px takes `paletteSwap` from the rule, `currentColor` from
`defaults`, and `addCss` from `defaults`' `16px` bucket. A rule that wants one substitution changed
says only that, and inherits the rest.

Suppressing an inherited value is the counterpart of declaring one. For a color, `"reset()"` is
already the expression that resolves to nothing, so `currentColor: "reset()"` leaves the icon
uncolored. For the others the empty declaration does the same: `paletteSwap: {}` swaps nothing,
`addCss: ""` injects nothing and `addCssClasses: []` adds nothing, each distinct from omitting the
key, which inherits. If multiple variants match with different CSS rules - the CSS (or classes) is
concatenated so everything is applied.

### Workbench fallbacks

A workbench describes where its own icons live by registering a manifest, from wherever it calls
`addIconPath` today:

```python
FreeCADGui.addIconRules(":/PartDesign/icons.yaml")
```

```cpp
IconManager::instance().addRules(":/PartDesign/icons.yaml");
```

The manifest uses the format already described. Only `icons`, `rules` and `defaults` are read from
it; `missing` belongs to the icon theme and is ignored, with a warning, if a contributed manifest
declares one. A workbench naming a handful of its icons exactly is the expected use of `icons`
there.

Everything a contributed manifest declares forms a tier strictly below everything the icon theme
says, its own `defaults` included - a contributed `icons` override outranks that manifest's own
rules, but never anything the theme provides. A workbench can therefore fill gaps but never override
the theme the user chose, whatever priority it writes. Within the tier, rules sort by `priority` and
then by the order their manifests were registered.

Because a rule applies only when it finds a file, a workbench has two ways to describe its icons. It
may name them, matching the family it owns; or it may point a catch-all at its own resource
directory and let existence do the filtering, which is the closest equivalent of the flat
`addIconPath` it replaces, but scoped so that the directory is consulted for its own icons rather
than for every icon in the application:

```yaml
rules:
  - match: "PartDesign_(.+)"
    searchPaths: [":/PartDesign/icons/$1.svg"]

  - match: "(.+)"
    priority: -10
    searchPaths: [":/PartDesign/icons/$1.svg"]
```

When a contributed rule supplies the file, `processing` resolves from that rule, then from its own
manifest's `defaults`, and then from the icon theme's. A workbench icon therefore still takes the
theme's stroke color and stroke weights unless the workbench deliberately says otherwise, which is
what keeps a contributed icon from looking foreign among themed ones.

`addIconPath` and `BitmapFactoryInst::addPath()` keep working and keep their meaning. They remain
the way to make a directory visible to the shipped catch-all templates, and a workbench that is
content with that need do nothing.

### Resolution pipeline

A request is a name, an optional size, a device pixel ratio, a `QIcon::Mode`/`QIcon::State` pair,
and an optional color override. It resolves in five stages.

1. The `icons` override for the name is tried first, then the rules in priority order - for each
   whose `match` accepts the name, its size bucket's templates and then its own are expanded and
   tested for existence - and `defaults` last. A candidate that is an alias rather than a path
   restarts this stage for the name it gives, keeping a record of the names already tried so a cycle
   ends in a warning rather than a hang. The first file found settles both the icon and the entry
   that supplied it. `QFile::exists` resolves both `:/` resources and the `icons:` Qt search path
   that `BitmapFactoryInst::addPath()` populates, so templates reach module- and addon-registered
   directories without the resolver enumerating them.
2. If nothing is found, the external icon theme directory (`Bitmaps/ExternalTheme`, honoring
   `PreferExternal`) and then the XDG/Qt icon theme (`UseIconTheme`) are consulted, in the order
   those preferences describe. These remain coded stages rather than declarable rules, being
   directory-scanning lookups driven by their own preferences rather than path templates.
3. Failing both, the `missing:` icon is resolved through stage 1 once and non-recursively. A
   `missing:` that does not itself resolve yields a null icon and one warning.
4. The format is taken from the resolved file's extension, and the processing for that format is
   resolved key by key from the applied entry down to `defaults`.
5. The five processing steps run in the order given under [Processing icons](#processing-icons), and
   the result is rasterized at `size × devicePixelRatio`. Every expression evaluated along the way
   sees the request's parameter overrides.

Stages 1 through 4 depend only on the theme file and the request. They perform no painting and,
apart from the existence tests in stage 1, no I/O.

### Units

`Gui::IconTheme` is the parsed manifest: pure data and a loader, with no I/O and no painting. It
includes no Qt header at all, so it can be built and tested on its own.

```cpp
/// What one layer says about processing. An unset member is inherited from the next layer down;
/// an empty one suppresses what would have been inherited.
struct SvgProcessing {
    std::optional<std::map<std::string, std::string>> paletteSwap;  // source color -> expression
    std::optional<std::string> currentColor;                        // what currentColor resolves to
    std::optional<std::vector<std::string>> addCssClasses;          // added to the document root
    std::optional<std::string> addCss;                              // placeholders substituted
};

/// What a rule, an override or `defaults` says, before any size refinement.
struct IconLayer {
    std::vector<std::string> searchPaths;   // templates; {} and $0..$n
    std::optional<SvgProcessing> processing;
};

/// A layer together with the blocks that refine it. `defaults`, every rule and every override has
/// this shape, and neither kind of block nests further.
struct IconEntry {
    IconLayer base;
    std::map<int, IconLayer> sizes;             // keyed by pixel size; one applies
    std::map<std::string, IconLayer> variants;  // keyed by name; any number apply
};

struct IconRule {
    std::regex match;
    int priority = 0;
    IconEntry entry;
};

/// An entry of the `icons` map. A plain string in the YAML sets both members, which is what makes
/// such a value a file when there is one and a redirect when there is not.
struct IconOverride {
    std::optional<std::string> alias;   // a name, resolved again from the top
    IconEntry entry;
};

enum class CandidateKind { File, Alias };

/// One thing to try, and the entry that proposed it.
struct IconCandidate {
    std::string value;                  // a path to test, or a name to resolve again
    CandidateKind kind;
    std::size_t source;
};

class GuiExport IconTheme
{
public:
    std::map<std::string, IconOverride> icons;  // exact-name overrides, ahead of every rule
    std::vector<IconRule> rules;                // sorted by priority when loaded
    IconEntry defaults;                         // applies when no rule does

    static IconTheme fromYaml(std::string_view yaml, std::string_view baseDirectory);
    static IconTheme fromFile(std::string_view path);

    /// Every path to try for @p name at @p pixelSize, in the order rules are considered.
    /// Which of them exists is the caller's business.
    std::vector<IconCandidate> candidates(std::string_view name, int pixelSize) const;
    /// The processing for the rule that proposed @p candidate, falling back to `defaults`.
    std::optional<SvgProcessing> svgProcessing(const IconCandidate& candidate, int pixelSize) const;
    std::string missingIcon() const;
};
```

The overrides are held as a map rather than as rules with literal patterns, so an exact name costs a
lookup instead of a walk through every pattern in the manifest.

Splitting the query in two is what keeps the theme free of file system access while still binding
the processing to the entry that supplied the file: `IconManager` walks the candidates, tests each
for existence, and asks for the processing of the one that won.

`Gui::IconManager` is the front door and the only stateful unit.

```cpp
struct IconMeta
{
    std::string iconId;
    std::string svgPath;
    bool themed = true;
};

struct RenderRequest {
    std::optional<QSize> size;                    // unset: the file's natural size
    qreal dpr = 1.0;
    std::optional<QColor> color;                  // unset: the theme's own color expressions
    std::vector<std::string> variants;            // which `variants` blocks of the manifest apply
    StyleParameters::ParameterValues parameters;  // scoped values for this render's expressions
    QIcon::Mode mode = QIcon::Normal;
    QIcon::State state = QIcon::Off;

    RenderRequest& withSize(int value);
    RenderRequest& withSize(QSize value);
    RenderRequest& withDpr(qreal value);
    RenderRequest& withColor(QColor value);
    RenderRequest& withVariants(std::vector<std::string> value);
    RenderRequest& withParameters(StyleParameters::ParameterValues value);
    RenderRequest& withMode(QIcon::Mode value);
    RenderRequest& withState(QIcon::State value);
};

QIcon   icon(std::string_view name, const RenderRequest& request = {});
QIcon   iconFromFile(std::string_view path, const RenderRequest& request = {});
QPixmap pixmap(std::string_view name, const RenderRequest& request);
QPixmap pixmapFromFile(std::string_view path, const RenderRequest& request);

/// Re-renders an already produced pixmap from its source under @p request.
const IconMeta* metaForPixmap(const QPixmap& pixmap) const;
QPixmap render(const IconMeta& meta, const RenderRequest& request) const;

void reload();  // re-read the theme file, drop every cache
void clear();   // drop rendered pixmaps only
```

Names, paths and expressions are standard library strings throughout, and a Qt type appears only
where the thing being described is itself a Qt one: an icon, a pixmap, and the members of a render
request, every one of which is handed straight to Qt to rasterize with. Nothing in the manifest
layer needs Qt to express it, so nothing there uses it.

A name and a path are distinct arguments to distinct methods rather than one argument whose shape
decides its meaning. `icon()` takes a request as well, of which it uses only the members that
outlive a single rendering - the color, the variants and the parameter overrides - because size,
mode and state are settled by Qt when the engine is asked to paint. An unset `size` means the
natural size, which is 64×64 for an SVG and the file's own dimensions for a raster image; this
reproduces what `BitmapFactoryInst::pixmap()` returns today.

`Gui::ParametrizedIconEngine` is the `QIconEngine` the manager hands out. It holds an icon name or a
verbatim path and nothing resolved, asking the manager on every call, so a theme change alters what
an already-issued `QIcon` paints without anyone re-issuing it. Each pixmap it produces is registered
with the manager against the metadata it came from, which is what makes the recovery described under
[Processing icons](#processing-icons) possible. It overrides `pixmap()`, `scaledPixmap()`,
`actualSize()`, `iconName()`, `key()`, `clone()` and `isNull()`.

`Gui::BitmapFactoryInst` gains the processing primitives as stateless members beside its existing
`pixmapFromSvg()`: apply a color map to SVG bytes, swap `currentColor` strokes and fills, scale
stroke widths under a selector, and rasterize at a size and device pixel ratio. The division of
labor is that `BitmapFactory` owns the mechanics and `IconManager` owns the decisions.

### Processing icons

Processing turns the resolved file into the document that is rasterized. Four steps are defined for
`svg`, and they run in a fixed order:

1. `paletteSwap` maps colors appearing literally in the source document to expressions.
2. `currentColor` names the color the document is drawn in.
3. `addCssClasses` adds class names to the document root.
4. `addCss` injects a stylesheet into a `<style>` element.

The order is fixed rather than declarable. Palette swap maps source colors and so has to see the
document before anything else touches it, and the stylesheet is injected last so that every class
the earlier steps and the request itself have added is already present for it to select on.

No step names particular elements. Anything that applies to some elements and not others is said in
the injected stylesheet, where a declaration overrides what the document carries as an attribute -
so a size that wants a lighter stroke declares `path, line { stroke-width: 1.5px; }`.

#### Variants

A request states which variants are active, and each names a `variants` block in the manifest whose
processing is layered over the rest, in the order given under [How a rule is
chosen](#how-a-rule-is-chosen). A variant is a named set of processing overrides that a call site
opts into.

```cpp
IconManager::instance().pixmap(
    "Part_Cut",
    RenderRequest {}.withSize({16, 16}).withVariants({"danger"})
);
```

With the manifest shown earlier, that icon takes `currentColor: "@Red.500"` from the `danger` block
instead of the `@BaseTextColor` it would otherwise inherit. A variant the manifest does not declare
is not an error and costs nothing: a theme answers the variants it knows about, which is what lets a
call site ask for one before any theme has an opinion about it.

Nothing obliges a variant to change color. The `muted` block in the same manifest adds a class and
leaves the stylesheet to say what that means; another could swap a palette entry outright. Adding a
class is one of the things a variant's processing may do, not what a variant is.

#### Adding classes

`addCssClasses` is a list of class names, added to the document root:

```yaml
addCssClasses: [fc-icon, muted]
```

They exist so that an injected stylesheet has something to select on that the artwork itself never
declared, and so that a variant can mark a document without saying what the mark means. Classes go
on the root and nowhere else; reaching an element inside the document is the stylesheet's job, by
element or by whatever classes the artwork already carries.

#### Injecting CSS

`addCss` holds a stylesheet, injected into a `<style>` element in the rendered document. Two
substitutions are performed on it first. Style parameter expressions are replaced by the values they
resolve to, under the same rules as every other value position in the manifest, so `stroke:
@Red.500` reaches the renderer as a literal color. And `currentColor` is replaced by the color the
icon is being rendered in, because Qt resolves `currentColor` from a `color` presentation attribute
but not from a CSS `color` declaration; substituting it in the text sidesteps that entirely.

Qt's SVG renderer applies a `<style>` element with class, element and descendant selectors, and a
declaration there beats the corresponding presentation attribute - which is what makes a stylesheet
able to override what the document already carries.

#### Style parameter overrides

A request may carry `StyleParameters::ParameterValues`, a map from parameter name to an
already-resolved value, which is in force for every expression the manifest resolves for that
render: in `paletteSwap`, in `currentColor`, and in the injected stylesheet.

Values are supplied per render rather than declared per widget, so they travel as scoped values, in
the manner an item already supplies its own to the style, rather than as a declared override set.
Supplying a value the loaded theme references nowhere costs nothing, and a value two call sites
supply alike is resolved once for both.

#### Colors supplied by the caller

A render request may also carry a color outright. It takes the place of `currentColor` for that
render only and leaves `paletteSwap` alone, so a monochrome glyph follows the caller while an icon
with baked-in colors keeps them.

A caller holding the `QIcon` can request a color, variants and overrides directly. A caller holding
only a `QPixmap`, which is what Qt's styling entry points such as `QStyle::generatedPixmap()` are
given, recovers the icon's metadata from the pixmap's cache key through `metaForPixmap()` and
re-renders from the source file rather than filtering the bitmap it was handed. A pixmap the manager
did not produce is not found, and the caller proceeds as it otherwise would.

### Python API

`FreeCADGui` gains three functions. Two of them mirror the C++ surface:

```python
FreeCADGui.icon(name: str,
                color: str | QColor | None = None,
                variants: Sequence[str] | None = None,
                parameters: Mapping[str, object] | None = None) -> QIcon

FreeCADGui.pixmap(name: str,
                  size: int | tuple[int, int] | None = None,
                  color: str | QColor | None = None,
                  variants: Sequence[str] | None = None,
                  parameters: Mapping[str, object] | None = None,
                  mode: QIcon.Mode = QIcon.Normal,
                  state: QIcon.State = QIcon.Off) -> QPixmap
```

`icon()` returns a `QIcon` backed by `ParametrizedIconEngine`, wrapped for PySide through the
existing `PythonWrapper::fromQIcon()`. Because the engine resolves through the manager on every
call, such an icon renders at whatever size Qt asks of it and follows a later theme change without
being re-fetched. Size, mode and state are consequently not parameters here: they belong to a
particular rendering, not to the icon, and Qt supplies them at paint time.

`pixmap()` is where those parameters do apply, since asking for pixels settles all of them. An
omitted `size` means the natural size, as it does in C++.

`color` accepts a `QColor` or the string spelling of one, and overrides `currentColor` for that icon
exactly as the C++ request does. It does not accept a style parameter expression; deciding a color
from the theme is the subject of the separate FEP named under [Further Work](#further-work).

`variants` names the manifest's `variants` blocks that should apply, and `parameters` supplies
values for named style parameters, in force for every expression the manifest resolves for this
icon. Values are given as values rather than as expressions - a color, a length, a number - because
they are supplied rather than declared. Both are described under [Processing
icons](#processing-icons), and both apply to `icon()` as well as `pixmap()` because neither is a
property of one rendering:

```python
button.setIcon(FreeCADGui.icon("Part_Cut", variants=["danger"]))
label.setPixmap(FreeCADGui.pixmap("Std_Refresh", size=16,
                                  parameters={"CurrentColor": "#418fde"}))
```

A third function registers a workbench's own rules, as described under [Workbench
fallbacks](#workbench-fallbacks):

```python
FreeCADGui.addIconRules(manifest: _Pathish, /) -> None
```

The existing `FreeCADGui.getIcon(name)` is unaffected and keeps its present meaning. It returns a
pixmap-backed `QIcon`, rendered once at the natural size in a fixed color, and remains the right
thing for code that wants a snapshot rather than a live icon. It gains theme resolution, since it
reaches `IconManager` through `BitmapFactory`, but not theme reactivity. `addIcon`, `addIconPath`
and `isIconCached` are likewise unchanged, `addIconPath` included: registering rules is an addition
beside it, not a replacement for it.

### Caching and invalidation

| Cache             | Key                                                           | Dropped by            |
| --------- | ------------------------------- | ----------- |
| Resolution        | icon name + pixel size                                        | `reload()`            |
| Source bytes      | file path                                                     | `reload()`            |
| Rendered pixmaps  | name or path, size, dpr, mode, state, color, variants, scope bin | `reload()`, `clear()` |

The resolution cache stores misses as well as hits. Once `BitmapFactory` delegates, every icon
request in the application walks the rule list, and an unresolvable name would otherwise re-run
every pattern and stat the filesystem on each repaint.

There are two invalidation triggers, matching what actually changed. An interface theme change calls
`clear()`, from `Application::reloadTheme()` beside the existing `clearTokenCache()`; only rendered
pixmaps are invalid, since resolution does not depend on color. An icon theme preference change
calls `reload()`, wired through the `ParamHandlers` machinery in
`Application::initStyleParameterManager()`, and also clears `BitmapFactory`'s `xpmCache`, which
would otherwise keep serving the previous theme.

### Shipped defaults

`qss:icons/defaults.yaml` would reproduce present behavior exactly and would be the preference's
default value:

```yaml
defaults:
  searchPaths:
    - "{}"              # absolute path or full resource path, verbatim
    - "icons:{}"        # the name already carries an extension
    - "icons:{}.svg"
    - "icons:{}.png"
    - "icons:{}.xpm"

missing: "help-browser"
```

A FreeCAD with no additional icon theme installed would therefore render as it does today, and the
format would begin doing work only when a second theme file exists.

### Impact on existing features / subsystems

`Gui::IconTheme`, `Gui::IconManager` and `Gui::ParametrizedIconEngine` would be new classes, and
nothing outside them would have to change in order for them to exist.

`BitmapFactoryInst::pixmap(const char*)` would delegate wholly to `IconManager`, so its 287 call
sites would gain theme resolution and processing without being edited, and would keep the pixel
dimensions they receive today.

`BitmapFactoryInst::iconFromTheme()` would keep its own ordering. It consults the XDG icon theme
before FreeCAD's own icons when `UseIconTheme` is set, which is the reverse of stages 1 and 2, so
only its terminal `pixmap()` fallback would delegate. Two orderings would therefore coexist,
deliberately: `icon(name)` answers what the icon theme says, `iconFromTheme(name)` answers what the
desktop says if it says anything. Both are documented as such.

`BitmapFactoryInst` would additionally gain the SVG processing primitives described under
[Units](#units). Its existing members would be unchanged.

`FreeCADGui` would gain the three functions described under [Python API](#python-api), declared in
`ApplicationPy` and typed in `FreeCADGui.module.pyi`. No existing Python function would change
signature or meaning, and no workbench would be required to register anything: the seven modules
that call `addIconPath` today, and the several that call `BitmapFactoryInst::addPath()`, would keep
working untouched.

Installation would add `icons/*.yaml` to the `Stylesheets` CMake data target beside
`parameters/*.yaml`.

### Backwards Compatibility

Documents and addons would be unaffected. No file format or property would change, and the Python
API would be added to rather than altered: `getIcon`, `addIcon`, `addIconPath` and `isIconCached`
would keep their signatures and their meaning, and `icon()`, `pixmap()` and `addIconRules()` would
be new names beside them.

Behavior would be unchanged for a default installation, because the shipped `defaults.yaml` would
express the present lookup. The one visible change would be a fix: an unresolvable icon name would
be reported once rather than on every repaint, because resolution failures would be cached.

Addons that register icon search paths through `BitmapFactoryInst::addPath()` would continue to work
unchanged, because the shipped rules resolve through the `icons:` search path those calls populate.
An addon shipping icons under names a theme's patterns capture would have those icons themed, which
is the intent.

## Open Questions

- The resolution mechanism is quite advanced so it may take time to find icons. It would be good to
  have a cache warmup mechanism that loads all icons and caches the exact matches across program
  runs so it is only slow first time.

## Rejected Ideas

Icon rules could have lived inside the interface theme file, under a reserved `_icons:` key in
`qss:parameters/{Theme}.yaml`. That would keep a theme to a single file, but it couples two things
that vary independently, and it would put structured data inside a file whose every other nested map
is read as dotted token names.

A single `icon()` method could have inferred name from path by the shape of its argument, which
would mean the fewest call-site changes. It was rejected because it leaves a dispatcher deciding
semantics from the form of its input when the caller already knows which of the two it holds.

The 807 `sPixmap` command declarations could have been migrated in the same change. They need not
be: command icons are already resolved by name through `BitmapFactory`, so the delegation themes
them without any of them being touched.

## Alternatives

Node-matching processing steps were considered and dropped: a stroke adjustment naming the elements
it applies to, and class additions targeting particular nodes. Both need a selector language, and
neither XPath nor a CSS subset is available to borrow - Qt 6 removed `QXmlQuery` and `QDomDocument`
has no selector API - so each would mean writing and maintaining a parser. Saying the same thing in
an injected stylesheet costs nothing and is handled by the renderer.

The theme's expressions could have been resolved once at load time, with the `IconTheme` rebuilt on
every interface theme reload. That is simpler at render time, but it couples the lifetimes of the
two themes this proposal separates.

`IconTheme` could have been folded into `IconManager` as a private structure, saving a file. The
loader could then only be tested through a singleton and global state.

## Implementation

Nothing here needs a new dependency. yaml-cpp already parses the style parameter files;
`ParameterManager::evaluate()` is public and takes an expression string, which is what makes a color
in the manifest able to be an expression rather than a literal; `PythonWrapper::fromQIcon()` already
exists and is what `getIcon` uses today; and `QFile::exists` already resolves both `:/` resources
and the `icons:` search path that `BitmapFactoryInst::addPath()` populates, so the templates reach
addon-registered directories without the resolver enumerating anything. A prototype of the rendering
half, an SVG recoloring icon engine driven by a manager, has been running in the author's branch.

`addCss` and `addCssClasses` rest on what Qt's SVG renderer supports. It applies a `<style>` element
with class, element and descendant selectors, a declaration there beats the corresponding
presentation attribute, and `stroke-width` set in CSS takes effect. It resolves `currentColor` from
a `color` presentation attribute on the root or an ancestor, but not from a CSS `color` declaration,
which is why the injected stylesheet has `currentColor` substituted textually along with the
placeholders.

The manifest layer holds no Qt type and includes no Qt header, so rule ordering, alias resolution,
per-key processing and bucket selection can all be exercised without Qt at all, not merely without a
`QApplication`. That is worth preserving deliberately, because it is what keeps the part of this
feature with the most behavior in it cheap to test. Only the engine's contract and the rendering
itself need Qt.

Alias chains are bounded by remembering the names already visited rather than by a depth limit. A
chain that returns to a name it has seen is abandoned with one warning and the original name carries
on through its remaining candidates, so a mistake in a manifest costs one wrong icon rather than a
hang. A depth limit would additionally turn a legitimate long chain into a mystery.

Reloading has a sharp edge. Workbenches register their manifests once, as they load, and are never
asked again, so re-reading the icon theme must keep every contributed source. A reload that rebuilt
the rule list from the theme file alone would drop every workbench fallback the moment a user
changed icon theme, and the icons would disappear with no error anywhere.

An unset size in a request means 64×64 for an SVG and the file's own dimensions for a raster image.
That is not a chosen number: it is what `BitmapFactoryInst::loadPixmap()` produces today, and
reproducing it exactly is what lets the 287 existing `pixmap()` call sites be redirected without any
of them being examined.

Finally, the route by which a color override reaches code holding only a `QPixmap` is the pixmap's
cache key. The engine records each pixmap it produces against the file and processing behind it, and
`metaForPixmap()` recovers that. Qt's styling entry points are the reason this is needed rather than
an `iconName()` lookup: `QStyle::generatedPixmap()` is handed a pixmap and asked for a variant of
it, with no `QIcon` anywhere in reach.

The pieces are separable and can land in sequence - the processing primitives, then the manifest
layer, then the manager and engine on top of them, then the delegation that puts existing call sites
onto it - with each step useful and testable before the next. The author intends to carry the work.

## Further Work

This proposal provides the mechanism for recoloring but does not settle what color an icon should
actually take: following the interface theme's tokens, following the text color of the control the
icon sits on, and reacting to interaction state such as disabled or hovered. That is a policy
question touching `FreeCADStyle` and the design token system, and is left to a separate FEP.

A themeable icon set makes several further things possible that are out of scope here: shipping a
filled counterpart to the current outline set, per-workbench icon overrides, and a preferences page
for choosing among installed icon themes. The last is the natural follow-up, since this proposal
adds the preference but no interface for it.

## Changelog

### 0.3 - 2026-09-12

- Remove ability to target specific SVG nodes with processing rules to simplify the scope.

### 0.2 - 2026-09-11

- Extended scope with variants of icons.

### 0.1 - 2026-09-09

- Initial draft.

## License / Copyright

All FEPs are explicitly [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
