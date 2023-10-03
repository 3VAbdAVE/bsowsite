# Trying to make this as simple to follow as possible.

First, cd into your site's root directory, these will be relative to that.

1. Copy nonblog.toml from themes/nonblog to ./config.toml
2. Edit config.toml as needed. Each line's purpose is docuemnted in that file.
3. Load static images into ./static/img. There's a specific directory for your favicons.
4. If you need additional head tags, you can create partials/extra-head.html and not need to modify the theme
