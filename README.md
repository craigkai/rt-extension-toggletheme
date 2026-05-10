# RT-Extension-ToggleTheme

Adds a light/dark mode toggle button to the RT 6 navigation bar.

Clicking the `circle-half` icon switches Bootstrap 5's `data-bs-theme` attribute between
`light` and `dark` for the current session. The preference is stored per user via
`SetPreferences` so it survives page navigation and reloads.

Only users with the `ModifySelf` right see the button.

## Changes from the upstream extension

### 1. Removed Elevator-only restriction

Both callbacks in the original contain a theme guard that bails out unless the active
stylesheet starts with `elevator`:

```perl
my $style = RT->Config->Get('WebDefaultStylesheet', $session{CurrentUser});
return unless $style =~ /^elevator/;
```

This guard was removed so the toggle button appears regardless of which theme is active.

**Why:** As additional themes (Nord, Terminal, …) were introduced, the toggle would
silently disappear whenever a user switched away from Elevator. Every Bootstrap 5 theme
that uses `data-bs-theme` supports light/dark switching the same way, so there is no
reason to restrict the toggle to a single stylesheet.

### 2. Removed bundled JS and server-side handler

The upstream extension ships two additional files that are absent in our version:

- **`static/js/themes.js`** — client-side JS that toggles `data-bs-theme` on `<html>`
  and fires an AJAX call to persist the choice:

  ```js
  jQuery('#theme_toggle').on('click', function() {
      var newTheme = jQuery('html').attr('data-bs-theme') === 'dark' ? 'light' : 'dark';
      jQuery('html').attr('data-bs-theme', newTheme);
      jQuery.ajax({ url: RT.Config.WebPath + '/Helpers/Toggle/Theme' });
  });
  ```

- **`html/Helpers/Toggle/Theme`** — the Mason endpoint that receives the AJAX call and
  writes the new mode to the user's RT preferences (`WebDefaultThemeMode`).

Both were removed because the active theme (Elevator, Nord, …) already provides its own
toggle implementation. Shipping a second copy would cause duplicate event handlers and
competing preference keys.

**Why:** Decoupling the button injection from the toggle logic means each theme can handle
dark/light switching in whatever way suits it best. Our version's callbacks do one thing:
put the `#theme_toggle` button into the navbar. The JS and persistence are the theme's
responsibility.

> **Note:** If you use a theme that does not implement `#theme_toggle` click handling, the
> button will appear but do nothing.

### 3. Self-service portal support

The upstream extension only injects the toggle into the **privileged interface**
(`PrivilegedMainNav`). We added an identical callback for the **self-service interface**
(`SelfServiceMainNav`).

**Why:** Self-service users spend as much time in RT as agents do and equally benefit from
choosing their preferred mode. Without the `SelfServiceMainNav` callback the toggle button
simply does not appear in the self-service portal, leaving those users stuck with whatever
the global default is.

Both callbacks apply the same `ModifySelf` guard, so self-service users who do not have
that right are unaffected.

### 4. Stripped package structure

The upstream extension is a full Perl package with `Makefile.PL`, `lib/`, `MANIFEST`, and
`inc/`. Our version contains only the Mason callback files and is deployed directly as a
local HTML override — no `make install`, no plugin registration required.

## How it works

Both callbacks inject the same HTML:

```html
<a href="javascript:" id="theme_toggle" class="nav-link menu-item">
  <!-- circle-half SVG icon -->
</a>
```

The `#theme_toggle` ID is picked up by JavaScript (provided by the active theme or RT core)
which toggles the `data-bs-theme` attribute on `<html>` between `light` and `dark` and
persists the choice.

## File structure

```
RT-Extension-ToggleTheme/
└── html/
    └── Callbacks/RT-Extension-ToggleTheme/Elements/Header/
        ├── PrivilegedMainNav   # Toggle for agents / admin interface
        └── SelfServiceMainNav  # Toggle for self-service portal (our addition)
```

This extension has no `Makefile.PL` or Perl module — it consists solely of Mason callbacks
and is deployed by copying the `html/` tree directly.

## Deployment

Copy the callback tree to the RT local plugin directory on the server:

```bash
sudo cp -r html/Callbacks/RT-Extension-ToggleTheme \
    /opt/rt6/local/html/Callbacks/
```

No plugin registration in `RT_SiteConfig.pm` is required — Mason picks up local callbacks
automatically.

Clear the Mason cache and restart the web server:

```bash
sudo systemctl stop apache2
sudo rm -rf /opt/rt6/var/mason_data/obj/*
sudo systemctl start apache2
```

## Related extensions

**RT-Extension-ThemeSwitcher** — lets users switch between different installed stylesheets
(e.g. Elevator, Nord, Terminal). ToggleTheme and ThemeSwitcher serve different purposes and
can be installed together: the ThemeSwitcher button (sort order 99.5) selects the
stylesheet, the ToggleTheme button (sort order 100) switches light/dark within that
stylesheet.

## Author

Torsten Brumm

## License

GNU General Public License version 2
