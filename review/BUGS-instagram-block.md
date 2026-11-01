# Bug review: instagram-block

Repo: EssentialBlocks/instagram-block (public). Fork: suhaeel/instagram-block
Branch: `review/instagram-block-bugs` (no changes to code, main untouched)
What it is: server-rendered Instagram feed block, version 1.2.0. The repo has no `dist/` (not built), so the plugin cannot be activated from a plain clone; I tested `includes/server.php` directly.

## What I ran
I loaded `eb_instagram_render_callback()` in the Local WordPress, fed it a **fake cached API response** (so nothing was sent to Instagram) and hostile values, and looked at the HTML it returns.

| Value | Where it ends up | Result |
|---|---|---|
| Caption `"><img src=x onerror=alert(1)>cap` | `<p>` and `alt="..."` | **unescaped in the output** |
| Username `user" onmouseover="alert(2)` | `href`, `alt=` (unquoted), link text | **unescaped, new attribute injected** |
| Block attribute `blockId` = `x" onmouseover="alert(3)` | `class` of the wrapper | **unescaped, new attribute injected** |
| Block attribute `profileImg` = `a.png onerror=alert(4)` | `<img src=...>` (unquoted) | **unescaped, `onerror` injected** |
| Block attribute `align` | `class` of the wrapper | not escaped either |

## Findings
| # | Severity | Where | What | Evidence |
|---|---|---|---|---|
| 1 | **High** | includes/server.php (render callback) | **Stored XSS.** Block attributes (`blockId`, `align`, `profileImg`, `profileName`) and Instagram API fields (`caption`, `username`, `permalink`) are concatenated into HTML without `esc_attr`, `esc_url` or `esc_html`. Anyone who can save block attributes, such as a Contributor, can put a script in a post that runs for every visitor, including admins. Block comment attributes are not filtered by `wp_kses`, so lacking `unfiltered_html` does not prevent it. | **Verified (ran it)** |
| 2 | **High (functional)** | includes/server.php:97 | The block uses `https://graph.instagram.com/me/media?...&access_token=...`. As far as I know Meta shut down the Instagram Basic Display API on 4 December 2024, so tokens for personal accounts no longer work and only Business or Creator accounts on the newer API do. I did not call the API. | From reading the code, not run, please confirm against Meta's current documentation |
| 3 | Medium | includes/server.php:97 | The token is put into the URL without `rawurlencode`. It is also saved in the block attributes (`token`), so it is stored in `post_content`, in revisions and in exports, and is readable by anyone who can open the post in the editor. | From reading the code, not run |
| 4 | Medium | includes/server.php:95-100 | A failed or error response is cached for 6 hours: `eb_instagram_add_to_cache($result)` runs whatever came back. An API error object (`{"error":...}`) is cached and the feed stays empty until it expires. There is also no timeout or HTTP status check in `wp_remote_get`. | From reading the code, not run |
| 5 | Medium | instagram-block.php:67-92 | Two `wp_register_script` calls use the same handle `eb-instagram-feed-block-script`. WordPress keeps the first one, so `dist/frontend/index.js` is never used and `assets/js/eb-instagram.js` runs. | From reading the code, not run |
| 6 | Low | includes/server.php:56 | Default `profileImg` is the hard-coded path `../wp-content/plugins/instagram-block/assets/images/user.png`, which breaks when the folder is named differently or WordPress is in a subfolder. | From reading the code, not run |
| 7 | Low | includes/server.php:130 | `date()` is used instead of `wp_date()`, so the site timezone and language are ignored. | From reading the code, not run |
| 8 | Low | includes/server.php:97 | `limit=500` is requested even when the block shows 6 images. | From reading the code, not run |
| 9 | Low | instagram-block.php:31 | `throw new Error(...)` when `dist/` is missing takes the whole site down. | From reading the code, not run |
| 10 | Info | Tested, not a bug | `usort` comparator uses `(int)(...)` so there is no PHP 8 "bool return" deprecation, and the order came out right on PHP 8.2. A 219-character cache key (long token) was stored and read fine here, so I could not reproduce a caching failure from long tokens. | **Verified (ran it)** |

## Suggested fix (not done)
Escape every value with `esc_attr` / `esc_url` / `esc_html` / `wp_kses_post` for the caption, quote all attribute values, and encode the token. Since this is a security fix to a public plugin, say so if you want it as a PR to the maintainers.

## Not covered
- Unless a row says "Verified", I only read the code. No automated tests were run.
- Chrome-only and WordPress 7.1.3 / PHP 8.2 on Local where something was run.

## Fix branch
Branch `fix/escape-output-and-api-handling`, pushed to the fork `suhaeel/instagram-block`. Based on the default branch; the default branch itself is untouched.

Escapes every value in the render callback (`esc_attr`, `esc_url`, `esc_html`, `sanitize_html_class`, quoted attributes), `rawurlencode`s the token, adds a request timeout and an HTTP status check, caches only a real `data` response, uses `wp_date`, and builds the default profile image URL from the plugin URL.

Tested: with a fake cached API response and the same hostile values as above, the output has 0 elements with `on*` attributes (before: injected in all 4 places) and a `javascript:` permalink is dropped. With normal data, the markup renders as before (links, new tab with `rel="noopener noreferrer"`, escaped caption, formatted date).
Not done: the Instagram API itself, the unbuilt `dist/`, and a decision on storing the token outside the block attributes.
