# Terminal.Gui 2.6 — Observed Specifics

Notes verified against package `Terminal.Gui 2.6.0-develop.61` and `Terminal.Gui.Editor 2.5.7`
from the `v2_develop` branch. Each item lists observed behavior, the root cause, and the working pattern.

## Application lifecycle

- Use `Application.Create()` to get an `IApplication` instance. The static `Application`
  facade still compiles but is deprecated.
- `IApplication` implements `IDisposable`. In F# bind it with `use` — `Dispose` performs
  `Shutdown` when the scope ends.
- Call sequence: `app.Init()` then `app.Run(runnable)`. `app.Init()` returns a value;
  add `|> ignore` to silence warning FS0020.
- Stop the session with `app.RequestStop(win)`, not with the static `Application.RequestStop()`.
- `Window` inherits `Runnable` (`Window -> Runnable -> View`). Pass the window to `app.Run()`.
- Headless tests: `app.Begin(win)` returns a `SessionToken`; finish with `app.End(token)`.
- `app.Navigation.GetFocused()` returns the focused `View`.
- `MessageBox.Query` takes the `IApplication` instance as the first argument:
  `MessageBox.Query(app, "Title", "msg", "OK")`.

## Key bindings model

Three binding levels exist:

- `View.KeyBindings` — commands invoke only when this view has focus.
- `View.HotKeyBindings` — commands invoke when this view is in the hierarchy but does not
  have focus. HotKeys cannot invoke invisible views. This rule is intentional design
  (see gui-cs/Terminal.Gui issue #2975: "Do not allow HotKeys to invoke a menu item if it
  is not visible. Shortcuts yes, hotkeys no.").
- `app.Keyboard.KeyBindings` — application-level bindings. Register with
  `AddApp(key, targetView, commands)`. The command invokes on `targetView` regardless of
  focus or visibility.

## MenuBar and MenuItem specifics

- `MenuBar` binds its activation key in the constructor. Set `MenuBar.DefaultKey` **before**
  `new MenuBar()`; assigning `menuBar.Key` later does not rebind. Default is `F10`.
- `MenuBarItem` children constructor takes `View[]`. In F# cast items: `exitItem :> View`.
- `MenuItem.Key` (e.g., `Key.Q.WithCtrl` in the constructor) lands in `HotKeyBindings`.
  The item lives inside a `PopoverMenu` that exists only while the menu is open. Therefore
  the key works only when the menu is open, not globally.

### MenuItem shortcut pitfalls

- `MenuItem.BindKeyToApplication <- true` fails silently if set before the item receives
  an `App` reference (`App` is null while the `PopoverMenu` is unattached). The setter
  removes the key from `HotKeyBindings` and calls `App?.Keyboard.KeyBindings.AddApp(...)`,
  which no-ops on null. The key ends up bound nowhere.
- Even when `App` is non-null, `BindKeyToApplication` binds `Command.HotKey`. A HotKey
  command does not reach `Action` on a hidden `MenuItem` (documented rule above).
- Working pattern for a global quit shortcut:

```fsharp
let exitItem = new MenuItem("_Exit", Key.Q.WithCtrl, Action(fun () -> app.RequestStop(win)))
app.Keyboard.KeyBindings.AddApp(Key.Q.WithCtrl, exitItem, [| Command.Activate |])
```

- `Command.Activate` on a `Shortcut`/`MenuItem` invokes its `Action`. This works even while
  the `MenuItem` is inside a closed `PopoverMenu`.
- Keep `Key` in the `MenuItem` constructor: it renders the shortcut hint in the menu and
  still works while the menu is open.
- `Command.Quit` on a `Runnable` requests stop. The default quit key is `Esc`
  (`Application.GetDefaultKey(Command.Quit)`); it works out of the box.
- Inside an open menu, key-initiated `Accept` converts to `Activate` in
  `Shortcut.OnAccepting` so the menu dismisses correctly.

## Editor (tui-cs/Editor, package Terminal.Gui.Editor)

- `Editor` replaces the deprecated `TextView`. Namespaces: `Terminal.Gui.Editor`,
  `Terminal.Gui.Editor.Document`, `Terminal.Gui.Editor.Highlighting`.
- Minimal setup:

```fsharp
let editor = new Editor()
editor.Width <- Dim.Fill()
editor.Height <- Dim.Fill()
editor.WordWrap <- true
editor.ViewportSettings <- ViewportSettingsFlags.HasScrollBars
editor.GutterOptions <- GutterOptions.LineNumbers ||| GutterOptions.Folding
editor.ConvertTabsToSpaces <- true
editor.HighlightingDefinition <- HighlightingManager.Instance.GetDefinitionByExtension(".fs")
editor.Document <- new TextDocument()
```

- `Editor` ships its own `KeyBindings` (`Ctrl+A` -> `SelectAll`, etc.). They work when the
  editor has focus. `OnKeyDownNotHandled` returns false for Ctrl/Alt/functional keys, so
  unhandled modified keys bubble up to the app-level dispatch.
- The first focusable subview receives focus at `Begin`/`Run`. `MenuBar` has
  `TabStop = TabGroup` and does not take focus, so the `Editor` gets focus automatically.

## Layout

- `Pos.Bottom(view)` anchors a view below a sibling; `Dim.Fill()` fills remaining space.
- Set `Y`, `Width`, `Height` on the editor; the `MenuBar` manages its own geometry.

## Testing hooks

- `Application.Create(new VirtualTimeProvider())` enables deterministic headless tests.
- `app.GetInputInjector()` returns `IInputInjector` (`Terminal.Gui.Testing`):
  `InjectKey(key, options)` then `ProcessQueue()`, then `app.RaiseIteration()`.
- `VirtualTimeProvider.Advance(TimeSpan)` moves virtual time.
- Input injection exercises the real binding dispatch: app-level `AddApp` bindings fire,
  `HotKeyBindings` fire, focused `KeyBindings` fire.

## Known issues in the develop build

- Issue #3908: top-level menu shortcut keys broken in some `v2-develop` builds.
- Issue #2975: menu rework still has open TODOs around key binding updates.
- The `BindKeyToApplication` silent-failure case is not documented; treat it as an
  unfinished area and prefer explicit `app.Keyboard.KeyBindings.AddApp(...)`.

## Documentation links

- https://gui-cs.github.io/Terminal.Gui — docs home (mirror: tui-cs.github.io/Terminal.Gui)
- https://gui-cs.github.io/Terminal.Gui/docs/menus.html — Menus Deep Dive
- https://tui-cs.github.io/Terminal.Gui/docs/application — Application Architecture
- https://github.com/gui-cs/Terminal.Gui — source, branch `v2_develop`; UICatalog is the
  reference app
- https://github.com/tui-cs/Editor — Editor package, Quickstart and `ted` sample
- Local API docs: `~/.nuget/packages/terminal.gui/2.6.0-develop.61/lib/net10.0/Terminal.Gui.xml`
