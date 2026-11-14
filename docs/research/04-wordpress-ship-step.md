# 04 · The Ship step on a WordPress site

Researched 2026-11-14. WordPress stable was 7.1.4 and 7.2 was at Beta 5 (https://wordpress.org/news/category/releases/). The SEO skills were read from the local clone at commit `805c9c8`. Documentation pages were read through a fetch tool that extracts quotes, and a few quotes are cut short where a page interleaves glossary pop-ups. Source code was downloaded and read directly. Nothing was run against a live WordPress site.

Shorthand in citations:

- `CORE/` is https://github.com/WordPress/wordpress-develop/blob/trunk/src/wp-includes/ (trunk, version string `7.2-beta5`).
- `YOAST 28.8` is the Yoast SEO package from https://downloads.wordpress.org/plugin/wordpress-seo.latest-stable.zip.
- `RM 1.0.282` is the Rank Math SEO package, https://plugins.svn.wordpress.org/seo-by-rank-math/trunk/.

## The question

For a WordPress Site, which of the changes the SEO skills make can be carried out through WordPress's own interfaces, and how should a Bot reach them?

## Short answer

**Use a mix, with the WordPress REST API as the base, authenticated by an application password for a dedicated WordPress user on each Site, and surfaced to Bots as our own governed MCP tools.** Use the Site's SEO plugin for SEO fields, and keep the Bot's browser for preview and for the few wp-admin screens that have no API. Do not make a WordPress-side MCP server the base for v1.

The main reason: REST with an application password is the only route that works on a self-hosted WordPress from 5.6 with HTTPS and nothing installed, and the only one with a plugin-free way to stage work (drafts for new content, autosaves for published content). The official MCP route is not ready to carry content writes: core's abilities are read-only through 7.2 Beta 5, the MCP Adapter is at 0.7.1, and its default server puts every ability behind one `execute-ability` tool, which our read-or-write classification cannot split.

How the intended flow maps:

- **New post or page.** First Approval saves it with `status: draft` (or `pending`). A second Approval sets `status: publish`.
- **Edit to a published post or page.** First Approval saves an autosave, which WordPress stores as a revision and does not show to visitors. A second Approval sends the same content as an update, which is live at once and leaves a revision to roll back to.
- **SEO fields, redirects, menus, settings.** These have no draft state in WordPress. The proposed values have to wait in our product, one Approval applies them live, and rollback is writing the old values back.

Three limits shape the design:

1. **A WordPress role cannot separate staging from applying on published content.** Saving an autosave needs the same permission as updating the post, and updating a published post without changing its status never checks the publish capability (section 3). The gate has to be our Approval, with the write credential held by the platform's MCP server and not on the Bot's computer. Name the tools that apply a change with a first verb OpenBot already asks on, such as publish or delete (note 02, section 3).
2. **SEO title, meta description, canonical, noindex, schema and redirects are not WordPress core.** They belong to whichever SEO plugin the Site runs, each with a different write path, and Rank Math has no documented one for a single post (section 4).
3. **Template, layout, new tool routes and server behaviour are out of reach** on a classic theme or a hosted Site. Much of what `seo-roast`, `block-ai-scrapers`, `domain-migration` and the tool-building skills ship cannot be a Ship step on a CMS site in v1. It becomes a Deliverable for the Client's developer.

## What the SEO skills change, and the WordPress route

| Kind of change | SEO skills that make it | WordPress route | Staged before live? |
|---|---|---|---|
| Title tag and meta description | seo-title-optimization, seo-autopilot, blog-refresher, blitz-seo, content-coverage, vs-and-alternatives, seo-roast (title templates) | SEO plugin field, per plugin (section 4). The visible post title is core REST `title` | No. Live on write |
| Edit or refresh an existing body | blog-refresher, unslop-seo, proof-kb (upgrade, route-fact), seo-autopilot, blitz-seo, intro, content-coverage (rewrite), youtube-seo (embed) | Core REST: autosave, then update | Yes, as an autosave |
| New post | content-coverage, seo-autopilot, proof-kb (draft), diff-marketing, parasite-and-pr (the canonical report) | Core REST `POST /wp/v2/posts` | Yes, `draft` or `pending` |
| New page (comparison, alternative, listicle, hub or index page) | vs-and-alternatives, ahrefs-research, dataforseo-serp, seo-autopilot, internal-linking (hubs) | Core REST `POST /wp/v2/pages`. A URL like `/vs/x` needs a parent page `vs` | Yes, `draft` or `pending` |
| Internal links inside existing content | internal-linking, blitz-seo, seo-autopilot, seo-roast, vs-and-alternatives, ahrefs-research, awesome-tools (badge link) | Same as a body edit | Yes, as an autosave |
| Images with alt text, cover or social image | seo-blog-images, content-coverage, blog-refresher | Core REST `POST /wp/v2/media` with `alt_text`, then `featured_media` or an image block in the body | The file is public at its URL on upload. Its use in a post is staged |
| Structured data (JSON-LD) | youtube-seo (VideoObject), internal-linking (BreadcrumbList), seo-roast (Product, Offer), domain-migration (Organization), block-ai-scrapers (Dataset), seo-autopilot | SEO plugin schema settings where it has a write path. A raw `<script>` in the body needs `unfiltered_html` | Plugin field: no. In the body: yes |
| Redirects | domain-migration, seo-autopilot, seo-roast, blog-refresher, dataforseo-serp, content-coverage | Not core. SEO or redirect plugin: API in some, wp-admin screen in others (section 4) | No |
| Canonical, noindex, hreflang | seo-roast, domain-migration, seo-autopilot, block-ai-scrapers | SEO plugin field. hreflang needs a multilingual plugin, not covered here | No |
| robots.txt, llms.txt, sitemap | seo-roast, block-ai-scrapers, domain-migration, seo-autopilot | robots.txt: SEO plugin screen in the browser. Sitemap: automatic. llms.txt: plugin toggle | No |
| Navigation and footer links | vs-and-alternatives, seo-roast, blitz-seo, internal-linking | Core REST `menu-items` (classic menus) or `navigation` (block themes), Administrator only | No |
| Templates, layout, components | seo-roast (above-the-fold layout, server-rendered pricing, badge), internal-linking (callout component), youtube-seo (embed component) | Block themes only: core REST `templates`, Administrator only. Classic themes: file access | No |
| New tool route or app code | dataforseo-serp, seo-autopilot (free tool), ahrefs-research, seo-roast (interactive element) | None. Needs plugin or theme code deployed | Not possible in v1 |
| Server or edge behaviour (middleware, firewall rules, headers, host-level redirect config, base URL) | block-ai-scrapers, domain-migration, seo-roast | None. Host or CDN control panel | Not possible in v1 |
| No Site change (off-site work, research files, drafts for a person to send) | directory-submissions, awesome-tools, linkdr, honest-link-outreach, reddit-serp, parasite-and-pr, aiseotracker, ai-seo-report, scribiz, understand-customers, customer-journey, blitz-copy, and the `seo/` working files of seo-autopilot and proof-kb | None. Files stay on the Bot's computer | n/a |

Source for the skills column: the `SKILL.md` files, for example `seo/seo-autopilot/SKILL.md:27` ("Everything you ship is a real change to the site: a new page, an edit, a redirect, a tool"), `seo/seo-roast/SKILL.md:231-233`, `seo/domain-migration/SKILL.md:293-300`, `seo/internal-linking/SKILL.md:249-256`, `seo/vs-and-alternatives/SKILL.md:164-197`, `content/seo-blog-images/SKILL.md:159-170`, `content/youtube-seo/SKILL.md:207-215`.

## Facts

### 2. WordPress core REST API

- **Posts and pages can be created and updated, and the status is a field.** https://developer.wordpress.org/rest-api/reference/posts/ lists `POST /wp/v2/posts` and `POST /wp/v2/posts/<id>`, and for `status`: "A named status for the post." "One of: `publish`, `future`, `draft`, `pending`, `private`". Pages have the same shape at `/wp/v2/pages` (https://developer.wordpress.org/rest-api/reference/). In core since 4.7 (`CORE/rest-api/endpoints/class-wp-rest-posts-controller.php:7`, `@since 4.7.0`).
- **`draft` and `pending` need no publish right, `publish` does.** `CORE/rest-api/endpoints/class-wp-rest-posts-controller.php:1570-1592`: `case 'draft': case 'pending': break;` and, for `publish` and `future`, `if ( ! current_user_can( $post_type->cap->publish_posts ) )` returns `rest_cannot_publish`, "Sorry, you are not allowed to publish posts in this post type."
- **Revisions can be listed, read and deleted over REST, not restored.** https://developer.wordpress.org/rest-api/reference/post-revisions/ lists `GET /wp/v2/posts/<parent>/revisions`, `GET /wp/v2/posts/<parent>/revisions/<id>` and `DELETE /wp/v2/posts/<parent>/revisions/<id>`. No restore route is on the page.
- **Autosaves can be created over REST** (since 5.0). Same page: `POST /wp/v2/posts/<id>/autosaves`. How they behave is in section 7.
- **Media upload and alt text.** https://developer.wordpress.org/rest-api/reference/media/ lists `POST /wp/v2/media` and the field `alt_text`: "Alternative text to display when attachment is not displayed." The file goes in the request body with a filename header: `CORE/rest-api/endpoints/class-wp-rest-attachments-controller.php:1996`, "Content-Disposition needs to be formatted as `attachment; filename="image.png"` or similar." Uploading needs `upload_files` (same file, line 411: "Sorry, you are not allowed to upload media on this site."). A post's featured image is the post field `featured_media`: "The ID of the featured media for the post."
- **Categories and tags** are post fields ("The terms assigned to the post in the category taxonomy.") with their own routes `/wp/v2/categories` and `/wp/v2/tags` (https://developer.wordpress.org/rest-api/reference/).
- **Custom post types appear only if the Site opted them in.** https://developer.wordpress.org/rest-api/extending-the-rest-api/adding-rest-api-support-for-custom-content-types/: "Setting this argument to true will add a route in the `wp/v2` namespace." (the argument is `show_in_rest`).
- **Custom fields appear only if registered for REST.** https://developer.wordpress.org/rest-api/extending-the-rest-api/modifying-responses/: "By setting `'show_in_rest' => true` when registering a new meta key, that key will be accessible through the REST API."
- **Not in core: an SEO title or a meta description.** Core prints only `<title>` from the post title and site name (`CORE/general-template.php:1562`, `echo '<title>' . wp_get_document_title() . '</title>'`). A search of that file finds no `<meta name="description">` output. The settings route has 20 fields, among them `title` ("Site title.") and `description` ("Site tagline."), and none for search visibility, robots.txt, permalinks or SEO (https://developer.wordpress.org/rest-api/reference/settings/).
- **Not in core: redirects.** None of the 21 resources in the endpoint reference is a redirect (https://developer.wordpress.org/rest-api/reference/).
- **Not in core REST: robots.txt.** It is generated on each request from the `blog_public` option and a PHP filter: `CORE/functions.php:1771`, `echo apply_filters( 'robots_txt', $output, $public );`. Neither is reachable over the settings route above.
- **Sitemaps need no Ship step.** https://make.wordpress.org/core/2020/07/22/new-xml-sitemaps-functionality-in-wordpress-5-5/: "With version 5.5., WordPress will expose a sitemap index at `/wp-sitemap.xml`." and "By default, sitemaps are created for all public and publicly queryable post types".
- **Correction to the brief: menus and block-theme templates are exposed, to Administrators.** https://developer.wordpress.org/rest-api/reference/nav_menu_items/ lists `POST /wp/v2/menu-items` (since 5.9, `CORE/rest-api/endpoints/class-wp-rest-menu-items-controller.php:7`). https://developer.wordpress.org/rest-api/reference/wp_templates/ lists `POST /wp/v2/templates` (since 5.8), gated by `CORE/rest-api/endpoints/class-wp-rest-templates-controller.php:184-187`: "Verify if the current user has edit_theme_options capability." A classic theme's PHP template files are not exposed.
- **Minimum versions that matter:** content routes 4.7, autosave route 5.0, sitemaps 5.5, application passwords 5.6, templates 5.8, menus 5.9, Abilities API 6.9. Current SEO plugins set a higher floor in practice: Rank Math 1.0.282 requires 6.7 (`readme.txt`), Yoast SEO 28.8 "Sets the minimum supported WordPress version to 7.0." (`YOAST 28.8`, `readme.txt` changelog).

### 3. Authentication and the Bot's WordPress role

- **Application passwords exist since 5.6 and are made on the user's profile.** https://developer.wordpress.org/rest-api/using-the-rest-api/authentication/: "As of 5.6, WordPress has shipped with Application Passwords, which can be generated from an Edit User page (wp-admin -> Users -> Edit User)." They are sent with Basic Auth: "The credentials can be passed along to REST API requests served over https://". An app can also send a logged-in person to an approval screen with `app_name`, `app_id`, `success_url` and `reject_url` (https://make.wordpress.org/core/2020/11/05/application-passwords-integration-guide/).
- **HTTPS is required by default.** `CORE/user.php:5195-5196`: "Application Passwords is supported only by sites using SSL or local environments but may be made available using the {@see 'wp_is_application_passwords_available'} filter." The same filter lets a Site or a security plugin switch them off.
- **They work for the API only, not for wp-admin.** `CORE/user.php:392`: "By default, Application Passwords is available for the REST API and XML-RPC." The integration guide on interactive login: "The point of application passwords are that they are to be used programmatically for applications, and not by humans for interactive sessions."
- **No scope narrower than the user.** Integration guide: "In future versions, the expectation is to include the ability to scope a given application password to limit its access." A request authenticated this way runs as that user (`CORE/user.php:514-519`, "Validates the application password credentials passed via Basic Authentication." returning "The authenticated user ID"). So the role of the Bot's WordPress user is the whole scope.
- **They can be revoked one by one.** Integration guide: "From the Edit User page, you can generate new, and view or revoke existing application passwords."
- **Roles** (https://wordpress.org/documentation/article/roles-and-capabilities/): Contributor is "somebody who can write and manage their own posts but cannot publish them", with `edit_posts`, `delete_posts`, `read`. Author adds `publish_posts`, `upload_files`, `edit_published_posts`, `delete_published_posts`. The capability table gives `edit_pages`, `edit_others_posts`, `edit_published_pages` and `unfiltered_html` (single site only) to Editor and above, and `edit_theme_options` and `manage_options` to Administrator.
- **What that means for a Bot account:**
  - **Contributor** can save new posts as `draft` or `pending` and nothing else we need: no media upload, no pages, no edits to published posts or to other people's posts.
  - **Author** can upload media but can also publish its own posts, so it is not least privilege.
  - **Staging an edit to someone else's published post needs Editor-level edit rights.** The autosave route reuses the update check (`CORE/rest-api/endpoints/class-wp-rest-autosaves-controller.php`, `create_item_permissions_check`: `return $this->parent_controller->update_item_permissions_check( $request );`), which is `current_user_can( 'edit_post', $post->ID )`. For a published post that maps to `edit_published_posts`, plus `edit_others_posts` when the Bot is not the author (`CORE/capabilities.php:253-268`).
  - **Those same rights allow a live update without the publish capability.** The publish check runs only when the request changes the status (`CORE/rest-api/endpoints/class-wp-rest-posts-controller.php:1343-1348`: `isset( $request['status'] ) && ( ! $current_status || $current_status !== $request['status'] )`).
  - **The smallest role that covers the table's content rows** is therefore a custom role with `read`, `edit_posts`, `edit_others_posts`, `edit_published_posts`, `edit_pages`, `edit_others_pages`, `edit_published_pages` and `upload_files`, without `publish_posts` and `publish_pages`. It stops the Bot publishing new content. It does not stop a direct update to live content. Our Approval has to do that.
  - **JSON-LD in a post body needs `unfiltered_html`.** The roles page: "Allows user to post HTML markup or even JavaScript code in pages, posts, comments and widgets." and "In WordPress Multisite, only Super Admins have the `unfiltered_html` capability." Without it content is filtered on save (`CORE/kses.php:3694-3695`).

### 4. The common SEO plugins

**Yoast SEO**

- **The vendor's REST documentation says read-only.** https://developer.yoast.com/customization/apis/rest-api/: "The Yoast REST API is currently read-only, and doesn't currently support `POST` or `PUT` calls to update the data." It adds `yoast_head` and `yoast_head_json` to core responses and offers `GET /wp-json/yoast/v1/get_head?url=`.
- **The shipped code has since added two write paths the page does not mention.** `YOAST 28.8`, `inc/class-wpseo-meta.php:101-121, 297-309`: `_yoast_wpseo_title`, `_yoast_wpseo_metadesc` and `_yoast_wpseo_focuskw` are registered with `'show_in_rest' => true`, `'object_subtype' => 'post'` and an `auth_callback` of `current_user_can( 'edit_post', $object_id )`. That makes them writable in the `meta` object of `/wp/v2/posts/<id>`, for posts only, not pages.
- **The second path is an ability, `yoast-seo/update-post-seo-data`.** `YOAST 28.8`, `src/abilities/user-interface/abilities/update-post-seo-data-ability.php`: its input covers `seo_title`, `meta_description`, `canonical`, `noindex`, `nofollow`, `schema_page_type`, `schema_article_type` and the social titles and descriptions. Changelog 28.7 (2026-10-13): "Allows agents to update titles and descriptions via the respective Yoast Ability." It needs `wpseo_edit_advanced_metadata`, which Yoast grants to `editor`, `wpseo_editor` and `wpseo_manager` (`admin/capabilities/class-register-capabilities.php:40`). It is reachable at `/wp-json/wp-abilities/v1/` and through the MCP Adapter (https://developer.yoast.com/features/yoast-seo-abilities/overview/).
- **Redirects are paid.** https://yoast.com/features/redirect-manager/: "Included in Yoast SEO Premium, Yoast SEO AI+, and Yoast WooCommerce SEO". No REST or WP-CLI documentation for redirects was found (see "Not verified").

**Rank Math**

- **Documented REST access is read-only.** https://rankmath.com/kb/headless-cms-support/: "Now you'll be able to send a GET request to our REST API endpoint for a specific URL." (`/wp-json/rankmath/v1/getHead?url=`, after enabling Headless CMS Support).
- **Its MCP tools do not write a single post's SEO fields or create redirects.** https://rankmath.com/kb/mcp-tools/ lists 27 tools. The post-level ones are reads (`rank-math/get-post-seo-meta`, `rank-math/get-post-schema`, `rank-math/get-redirections`). The writes are site settings (`rank-math/set-*`) and `rank-math/fix-site-seo`. The free package agrees (`RM 1.0.282`, `includes/abilities/`).
- **An undocumented write route exists.** `RM 1.0.282`, `includes/rest/class-shared.php:49-80` registers `POST rankmath/v1/updateMeta`, `/updateRedirection` and `/updateSchemas`, allowed for a user who can edit the post. It is the plugin's own editor plumbing, not a published API, so it can change without notice. No Rank Math meta key is registered for the core `meta` field (search of the package for `register_meta` and `register_post_meta` finds none).
- **Redirects are in the free plugin** (module `includes/modules/redirections/` in `RM 1.0.282`), managed at "Rank Math SEO → Redirections" (https://rankmath.com/kb/setting-up-redirections/). That page marks CSV import and export as PRO.

**All in One SEO**

- **Read and write over the core routes, free.** https://aioseo.com/docs/fetching-updating-aioseo-data-via-the-wordpress-rest-api/: "The REST API feature is available in All in One SEO Lite and All in One SEO Pro", and "you can update data via a POST/PUT request by adding the property `aioseo_meta_data` in the body of the request." Writes need one of AIOSEO's own capabilities, such as `aioseo_page_general_settings`. The page shows only `title` and `description` as examples.
- **Redirects are paid.** https://aioseo.com/docs/connecting-an-ai-assistant-to-your-site-with-aioseos-mcp-server/: "All in One SEO Pro adds abilities for terms, redirects and Search Statistics."

**SEOPress**

- **Read is public, write is per section.** https://www.seopress.org/support/guides/get-started-with-the-seopress-rest-api/: "These GET routes are public and do not require authentication or the `edit_post` capability." Writes are `PUT` on sub-routes such as `/wp-json/seopress/v1/posts/[postId]/title-description-metas`, with `edit_post`. The guide recommends application passwords.
- **Abilities, including redirects in PRO.** https://www.seopress.org/support/guides/seopress-abilities/: free has `seopress/update-post-title-description` and `seopress/update-post-robots-settings`. PRO adds `seopress/create-redirection`, `update-redirection`, `delete-redirection` and `update-post-schemas`. Needs WordPress 6.9 and SEOPress 10.0.

**A redirect plugin as a fallback.** The free Redirection plugin has `post /redirection/v1/redirect`: "Create a new redirect and return the current redirect list." It adds: "Most endpoints require a user with the relevant Redirection capability; in practice this often means an administrator." (https://redirection.me/developer/rest-api/). It is third-party and would have to be installed on the Client's Site.

### 5. What REST cannot do, and the realistic route on a hosted Site

- **Create the application password.** A person does this once in wp-admin (section 3). Realistic: the Operator or the Client, in a browser.
- **robots.txt.** Core has no route. Rank Math's editor is at General Settings, "Edit robots.txt", and a physical file blocks it: "you'll need to delete the actual robots.txt file (if any) from your website's root folder using an FTP client." (https://rankmath.com/kb/how-to-edit-robots-txt-with-rank-math/). Realistic: the SEO plugin's screen in the Bot's browser, behind an Approval.
- **Redirects on Yoast Premium and Rank Math Sites.** No documented write API. Realistic: the plugin's wp-admin screen in the browser, or install the Redirection plugin.
- **Classic-theme templates, layout, new routes, plugin code.** Needs files on the server. WP-CLI is not a way round it for us: https://make.wordpress.org/cli/handbook/guides/running-commands-remotely/ says it "proxies commands to the ssh executable" and "you need to have a copy of WP-CLI installed on the remote server, accessible as `wp`." Realistic: not in v1. Hand the Client's developer a Deliverable.
- **Installing a plugin** (MCP Adapter, Redirection). Core has `/wp/v2/plugins`, gated on `install_plugins` and `activate_plugins` (`CORE/rest-api/endpoints/class-wp-rest-plugins-controller.php:242-250`). Realistic: the Client's Administrator does it. A Bot should not hold that role.
- **Seeing a staged change in the Site's own theme.** Needs a browser session (section 7).
- **One-click restore of a revision.** wp-admin has it ("Click the Restore button to restore a revision.", https://wordpress.org/documentation/article/revisions/). Over REST the Bot reads the old revision and sends its content back as an update.
- **Server and edge behaviour.** Not WordPress. Realistic: none in v1.

### 6. MCP for WordPress

**Official WordPress project**

- **Abilities API, in core since 6.9.** https://developer.wordpress.org/news/2025/11/introducing-the-wordpress-abilities-api/: "The Abilities REST API endpoints are available under the `wp-json/wp-abilities/v1` namespace." Each ability has a permission callback: "A function that determines whether the current user has permission to execute the Ability."
- **Core's own abilities are read-only.** `CORE/abilities.php:52`: "@since 7.2.0 Added the `core/content-query`, `core/settings-get`, and `core/users-query` abilities.", and lines 374-381 call each "currently the read-only". Write abilities are not scheduled: "Additional single-item and write abilities remain under development and may follow after the initial release." (AI Contributor Weekly Summary, 14 October 2026, https://make.wordpress.org/ai/).
- **MCP Adapter.** https://github.com/WordPress/mcp-adapter: "The official WordPress package for MCP integration that exposes WordPress abilities as Model Context Protocol (MCP) tools, resources, and prompts for AI agents." Version 0.7.1 (2026-10-15), requires WordPress 6.9, 70,000+ installs (https://wordpress.org/plugins/mcp-adapter/). 0.7.0 was a breaking release. A 1.0 is proposed, not decided: the team "proposed targeting a stable MCP Adapter 1.0 release around December 8, alongside WordPress 7.2." (Weekly Summary, 28 October 2026).
- **Its default server exposes three tools, not one per ability.** `docs/guides/default-server.md` in that repository: "It exposes three meta-tools that let AI agents dynamically discover and execute abilities", namely `mcp-adapter-discover-abilities`, `mcp-adapter-get-ability-info` and `mcp-adapter-execute-ability`. Endpoint `/wp-json/mcp/mcp-adapter-default-server`. "Authentication: Requires logged-in WordPress user with `read` capability". In practice that is an application password.
- **Abilities are private unless marked.** README: "WordPress abilities are private by default. Set `meta.public` (or `meta.mcp.public`) to `true` to expose one".

**Automattic**

- **`Automattic/wordpress-mcp` is finished.** https://github.com/Automattic/wordpress-mcp: "This repository is deprecated and archived." and "This repository was archived by the owner on Sep 17, 2026."
- **WordPress.com MCP is the fullest tool set, for its own Sites.** https://developer.wordpress.com/docs/mcp/: "WordPress.com exposes an MCP server for AI agents, at https://public-api.wordpress.com/wpcom/v2/mcp/v1." "The WordPress.com MCP server uses OAuth 2.1 for authentication." "WordPress.com MCP is included on all WordPress.com paid plans." "Jetpack-connected self-hosted sites reach the same server with a Jetpack AI or Jetpack Complete plan".
- **Its tools** (https://developer.wordpress.com/docs/mcp/tools-reference/) include `posts.create` ("Create a new post — creates as **draft** by default."), `posts.update` ("If status is `publish`, changes go **live immediately**."), block-level `post-sections.*`, `media.create`, `media.update`, `revisions.restore`, `templates.*`, `menus.*`, `navigation.*` and `settings.update`. SEO coverage is site-level only (`advanced_seo_front_page_description`, `advanced_seo_title_formats`). No tool for a post's SEO title or for URL redirects was listed. "Write, update, and delete operations can require user confirmation, depending on the user's approval settings."

**SEO plugin vendors** (third-party to WordPress, first-party to their plugin). Yoast, Rank Math, All in One SEO and SEOPress all register abilities and rely on the MCP Adapter to serve them (section 4). All in One SEO: "The free MCP Adapter plugin is required too."

**Community servers.** Example: https://github.com/InstaWP/mcp-wp, "This is a Model Context Protocol (MCP) server for WordPress", which wraps the REST API with an application password. About 100 stars, no releases. Others exist and were not assessed.

**Coverage of the table.** The official route covers the SEO-field rows where the Site's plugin offers write abilities, and none of the content rows, because core has no write abilities yet. WordPress.com MCP covers the content, media, menu and template rows, only for WordPress.com and Jetpack-connected Sites, and not per-post SEO fields or redirects.

### 7. Safety properties for the Approval flow

- **New content stays private until published.** https://wordpress.org/documentation/article/post-status/: Draft is "Incomplete post viewable by anyone with proper user role." Pending is "Awaiting a user with the `publish_posts` capability (typically a user assigned the Editor role) to publish."
- **A change to a published post can be staged without a plugin, as an autosave.** https://wordpress.org/documentation/article/revisions/: "Autosaves are enabled for all posts and pages but do not overwrite published content. Autosaves are stored as a special type of revision; they do not overwrite the actual post." The REST route follows the same rule: `CORE/rest-api/endpoints/class-wp-rest-autosaves-controller.php:236-237`, "When a post is still in draft form, updates from the author can directly update the post. Other autosaves must be stored as per-user autosave revisions."
- **One autosave per user per post.** Same page: "There is only ever a maximum of one autosave per user for any given post. New autosaves overwrite old autosaves." A second staged edit to the same post replaces the first.
- **An autosave holds title, content and excerpt.** "By default, WP keeps track of the changes to title, author, content, excerpt." SEO plugin fields, categories and the featured image are not staged this way.
- **Do not stage by changing the status.** Setting a published post to `draft` or `pending` takes it off the public Site, since only `publish` is "Viewable by everyone." (inference from the status definitions, not tested).
- **Preview.** The autosave response carries `preview_link`, "Preview link for the post." (`class-wp-rest-autosaves-controller.php:353`), built with `preview_id` and `preview_nonce`. Opening it needs a logged-in browser session with edit rights: `CORE/class-wp-query.php:3511`, "User must have edit permissions on the draft to preview." The nonce is "tied to a specific action, user, user session, and window of time" (`CORE/pluggable.php:2535-2536`), and an application password does not create a browser session. So the link from the API will probably not open for the Operator. Two routes that should work: the Bot opens Preview from the editor in its own logged-in browser and takes a screenshot, or the Workbench shows a before-and-after diff built from the autosave's content read over REST. Core has no public share link for a preview.
- **Rollback of content.** Every update is kept: "The WordPress revisions system stores a record of each saved draft or published update." Restore is the wp-admin button or re-sending the old revision's content (section 5). The Site can limit or switch off revisions: `WP_POST_REVISIONS` set to "false, 0: do not store any revisions (except the one autosave per post)". The Bot should check a revision exists before relying on it.
- **Rollback of SEO fields.** Not in revisions. Yoast states the rule: "Any change made through the WordPress Abilities API can be rolled back" "as long as your AI tool holds the changed values in context." (https://yoast.com/yoast-seo-sept-29-2026/). Our Activity record must store the old value of every field a Bot overwrites.
- **Delete goes to Trash by default.** Posts reference, `force`: "Whether to bypass Trash and force deletion."

## Open risks and what could not be verified

Risks:

- **The Bot's browser is a second, ungoverned path to the same Site.** A wp-admin session has every right of its WordPress user and no read-or-write classification. If the Bot keeps a browser login for preview, give that user the lowest role that still previews, and keep the write credential in the platform's MCP server only.
- **Per-plugin adapters.** Four plugins, four write paths, and Rank Math's is undocumented. A Site with no SEO plugin has nowhere to put a meta description at all.
- **Yoast's documentation and code disagree** on whether writes exist. The code was read at 28.8 only. Older Sites will behave as the documentation says.
- **MCP Adapter is pre-1.0** and its one-tool default server does not fit a per-tool Approval rule. Using it for SEO fields means classifying by the ability name inside the call, or asking the Client to register a custom server. Whether OpenBot can add a server that authenticates with a username and application password was not checked here.
- **Page builders.** Sites built with Elementor, Divi and similar are understood to keep layout outside the post body, so a REST edit to `content` may not change what visitors see. From memory, not verified.
- **Caching.** A page cache or CDN may keep serving the old page after an update. From memory, not verified.
- **Application passwords can be switched off** by the filter in section 3. Which hosts and security plugins do so by default was not established (the Wordfence help page returned no content).

Not verified:

- Yoast Premium's redirect write path. The premium code is not public and `developer.yoast.com/features/wp-cli/redirects/` returned 404.
- Rank Math PRO's abilities, and whether PRO adds per-post writes.
- All in One SEO's individual ability names, the full list of writable `aioseo_meta_data` keys, and which paid tier includes redirects beyond "Pro".
- SEOPress `PUT` body field names, and whether the per-post redirection route is free.
- Whether a `<script type="application/ld+json">` block survives saving for a user without `unfiltered_html`. Inferred from the capability's description, not tested.
- The preview nonce behaviour. Inferred from source, not tested.
- How the block editor presents a Bot's autosave to a person who opens the same post.
- Whether typical Client hosts offer SSH and WP-CLI.
- Whether WordPress.com-hosted Sites accept application passwords. Only its MCP server was read.
