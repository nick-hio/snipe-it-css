# snipe-it-css

CSS redesigns for [Snipe-IT](https://github.com/snipe/snipe-it), installed through its built-in **Custom CSS** setting.

## Themes

| Theme | Style | Screenshots |
| --- | --- | --- |
| [Porcelain](themes/porcelain/) | Light and dark modes, cobalt accents and rounded controls | [Preview](themes/porcelain/README.md#screenshots) |

## Install

1. Open a theme's `custom.css` file, select **Raw**, and copy the entire file.
2. Sign in to Snipe-IT as an administrator. Open **Settings → Branding → Custom CSS**.
3. Back up any existing custom CSS, replace it with the theme, and save.
4. On the same page, enter the Branding colours from the theme's README, if it lists any, and save.
5. Refresh the page. If needed, refresh without the browser cache.

Paste the CSS directly, without a `<style>` wrapper. To remove a theme, clear the field or restore your previous CSS.

Snipe-IT limits Custom CSS to 65,535 bytes and cuts longer CSS without an error. To raise the limit, see [Raise the Custom CSS size limit](docs/custom-css-size-limit.md).

## Add another redesign

Keep each theme in a separate directory:

```text
themes/
  porcelain/
    custom.css
    README.md
    screenshots/
  your-theme/
    custom.css
    README.md
    screenshots/
```

Use a lowercase theme name with hyphens. Include installation notes, the Snipe-IT version tested, and screenshots made with sample data. Add the theme to the table above.
