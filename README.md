# Sticky Notes Board

`index.html` is the complete GitHub Pages-ready widget. It contains the HTML, CSS, and vanilla JavaScript in one file and loads the official Supabase JavaScript client from jsDelivr at the pinned version `2.109.0`. No application framework, build step, custom backend, database password, secret key, or service-role key is used.

No commit, push, amend, or tag operation was performed. To publish, place `index.html` at the root of the GitHub Pages publishing source, publish that source, and embed its HTTPS URL in Notion. A 568px embed height and width of at least 320px are recommended.

## Supabase and authentication

The client is initialized with the supplied project URL and publishable key. Auth enables `persistSession` and automatic token refresh, and disables URL-based session detection because this widget uses email/password only. Supabase stores and restores the authenticated session using its normal browser storage behavior.

The signed-out view contains Email, Password, and Sign in controls. It calls `signInWithPassword`, shows a generic user-safe error on failure, and replaces the form with the board after a valid session appears. Log out flushes pending note edits before calling Supabase sign-out. OAuth, magic links, social login, Notion authentication, and sign-up UI are absent.

The project health check returned HTTP 200, reported email authentication enabled, and returned an empty array for an unauthenticated `sticky_notes` SELECT. The widget did not make schema, policy, or RLS changes.

## Data loading and ownership

After a session is available, the widget selects these columns from `public.sticky_notes`:

```text
id, created_at, text, color, sort_order, rotation, visual_variant, user_id
```

The query includes `user_id = current authenticated user id`, orders by `sort_order` and then `created_at`, renders only validated rows for that user, and subscribes after the initial fetch finishes. A loading state remains visible until the fetch completes.

The browser never accepts a user ID from an input. Every insert obtains `user_id` from the current Supabase Auth session. Updates and deletes include both the row `id` and authenticated `user_id` filters. The existing RLS policies remain the final authorization boundary.

## CRUD and live synchronization

- Add inserts a complete row with empty text, the next `sort_order`, the authenticated `user_id`, a palette color, a deterministic rotation, and a deterministic `visual_variant`. The returned database row is then rendered and focused.
- Text updates immediately in the textarea, then saves 500ms after typing stops. Blur flushes the pending edit immediately. Failed saves retry after 2.5 seconds. Pending edits are also flushed on logout and attempted during page exit.
- Color changes optimistically in the selected note and updates only that row. A failed update restores the previous color.
- Delete waits for a successful user-scoped database deletion, removes the local note, and recalculates layout. A failed deletion leaves the note in place.

The Realtime channel listens to Postgres Changes on `public.sticky_notes` with the server-side filter `user_id=eq.<authenticated user id>`. Inserts are de-duplicated by database ID. Incoming changes update local state without issuing a write back, so Realtime cannot create echo loops.

When an UPDATE arrives for a textarea that is focused or has a pending local edit, its remote text is deferred. Other fields may update immediately. The current local text stays visible; after blur, a pending local save completes, or a deferred remote value is applied when there is no local edit. Realtime disconnect states appear unobtrusively in the footer. Returning focus to the widget triggers a source-of-truth refresh when no note is being edited.

## Fixed layout

The complete widget is 568px tall and fills the available width up to 960px. At widths above 420px, the board area is 464px tall. At narrower widths it is 468px tall because the outer padding is reduced. Note count never changes these dimensions.

Maximum: **12 notes**. At capacity, Add Sticky Note is disabled and the footer reads `Board full · 12 / 12`.

The layout tests candidate column counts and chooses the largest square note that fits the current board. Column caps follow the requested density:

| Note count | Maximum columns | 900px-wide note size |
| --- | ---: | ---: |
| 1 | 1 | 260px |
| 2 | 2 | 260px |
| 3–4 | 2 | 207px |
| 5–6 | 3 | 207px |
| 7–9 | 3 | 130px |
| 10–12 | 4 | 130px |

Narrow embeds may use fewer columns. Note padding scales between 7px and 22px; handwriting-style text scales between 12px and 21px. Deleting notes recalculates the grid and lets remaining notes grow when the available area permits.

## Visual system and motion

Available colors are warm yellow, pastel pink, soft blue, pastel orange, turquoise, lavender, coral, and sage green.

`sort_order` deterministically selects one of eight rotations from −0.9° to +0.8° and one of four stored decoration variants: pink tape, turquoise tape, orange tape, or a small pin. The supplied reference influenced the softly irregular corners, folded paper edge, pastel harmony, translucent tape, pins, gentle shadows, blue graph texture, faint ruled texture, and handmade stationery character. Its layout and individual note designs were not copied.

Only a successfully inserted new note receives the 220ms entry animation: opacity fades in, scale moves from 0.94 to 1, and the paper settles downward by 5px while retaining its rotation. Existing notes do not replay it. Reduced-motion users receive no entry animation or transitions.

## Notion transparency and accessibility

`html` and `body` use zero margin and padding, 100% dimensions, and `background: transparent !important`. The widget wrapper and board do not paint a page-wide surface. Only the login note, sticky notes, compact controls, and palette have backgrounds. Light and dark preferences adjust interface colors without hardcoding a Notion page background.

The widget uses labeled form fields, accessible names for add/color/delete actions, numbered textarea labels, visible focus rings, keyboard-editable plain-text areas, arrow-key palette navigation, contained Tab navigation in the palette, and Escape dismissal. Touch devices keep note controls visible.

## Validation results

Automated headless Edge tests passed for:

- signed-out state, incorrect password, valid sign-in, session restoration, logout, and login again;
- create, 500ms debounced edit, color change, delete, refresh, and user-scoped mutation filters;
- two open windows receiving inserts, text updates, colors, and deletes without duplicate notes;
- protection against a remote text update while a local textarea is active;
- 1, 2, 4, 6, 9, and 12-note layouts, capacity disabling, and delete-to-grow reflow;
- fixed board measurements, transparent outer surfaces, no overlapping note rectangles, no page overflow, and 320px narrow-embed containment;
- short add-note animation, reduced-motion behavior, Realtime disconnect messaging, and failed-insert handling;
- successful loading of the pinned official Supabase client from its production CDN with no browser runtime or network errors.

The live project was checked without credentials: the API and auth settings endpoints were reachable, email auth was enabled, and an unauthenticated table request returned no rows. Valid-account login, authenticated CRUD, session restoration, and live two-device synchronization could not be exercised against the real project because no test account credentials were supplied; those flows were tested against a deterministic browser mock implementing the same client surface.

## Remaining limitations

- The Supabase Auth session depends on browser storage permitted inside the Notion iframe. Browser privacy settings may partition or block that storage; signing in again restores access to the same server-side notes.
- GitHub Pages publication and the final Notion embed were not performed, as the request prohibits pushing. They remain environment-specific integration checks.
- Supabase documents that filtered Postgres Changes DELETE events require appropriate replica-identity support. This widget safely keeps the user filter. If the table is not configured to deliver filtered deletes, a deletion from another device will be reconciled when this widget regains focus rather than appearing instantly. No unsafe unfiltered DELETE subscription is used.
- The 12-note maximum is enforced by each client. Simultaneous additions from multiple devices could exceed it unless the database also enforces a per-user limit; the widget renders the first 12 ordered rows.
- Long notes retain their full database text, but only part fits visibly on very small paper. The board never scrolls or expands. At 320px width and high note counts, the widget is intended for brief reminders.
- Concurrent text edits use a simple last-successful-write model. Active local typing is protected from incoming visual replacement, but this is not a character-level collaborative editor.
- Deletion is immediate and has no undo. Handwriting typography varies with the fonts installed on the viewing device.
