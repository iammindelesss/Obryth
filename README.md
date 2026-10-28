# Obryth

Obryth is a software company site and 3D-printing platform made of plain HTML, CSS and JavaScript, hosted free on GitHub Pages, with accounts, the model library and file storage provided by Supabase.

- **Obryth Calculator**: filament, electricity and other costs, selling price, profit and margin.
- **Obryth Model Library**: browse, search, upload, favourite, comment on, report and download 3D models (STL, 3MF, OBJ, STEP).
- **Obryth Studio**: a separate desktop app. No release exists, so the site shows a Coming Soon page and no download.
- **User Profile**, public creator profiles, follows, notifications and **Settings**.

Premium listings can be created and shown with a price, but **payments are not set up**: nobody can buy, and premium files cannot be downloaded.

## Files (4 files + the `assets` folder on GitHub, 2 for Supabase)

| File | Purpose |
| --- | --- |
| `index.html` | All pages, including the homepage (hero, products, philosophy, closing call to action) |
| `style.css` | Styling (dark theme, palette at the top) |
| `assets/` | The Obryth logo in web sizes, favicons and the calculator preview (see "Logo and brand assets") |
| `script.js` | Calculator, library, uploads, accounts, notifications, comments, profiles, page switching and homepage motion. Supabase settings are at the top. |
| `README.md` | This guide |
| `supabase-migration.sql` | Paste into the Supabase SQL editor. **Not uploaded to GitHub.** |
| `moderation-function.ts` | Pasted into Supabase Edge Functions. **Not uploaded to GitHub.** |

Links and files use relative paths, so the site works at its current address. The repository name and site address were not changed.

## Setup (in this order)

1. **Run the migration (the corrected one).** Supabase → **SQL Editor → New query**, paste all of `supabase-migration.sql`, click **Run**. It is safe to run again, never deletes data, and now creates everything in the right order (the previous version tried to use the `download_count` and `favourite_count` columns before adding them, so it could not finish and the new tables were never created). It builds comments, favourites, follows, download history, notifications, preferences and the reports columns, fixes the counters, and **replaces the row-level-security policies on those tables with a reviewed set** (policies made by hand cannot be inspected from outside). If it prints a NOTICE about duplicate usernames or reports, read it and run the file again after fixing them. Then run the check in "After running the migration" below.
2. **Upload the 4 site files and the whole `assets` folder** to GitHub (replace the old ones; in Add file → Upload files you can drag the `assets` folder in). Wait a couple of minutes and hard-refresh (Ctrl+Shift+R). The browser console (F12) should show `Obryth build 9`.
3. **Reports email (only if not done yet):** follow "Reports and removal" below. The function's emails now say "Obryth"; re-paste `moderation-function.ts` into your existing `moderation` function and redeploy to get the new wording.

**Because the repository was renamed, update Supabase → Authentication → URL Configuration:** set **Site URL** and add under **Redirect URLs** `https://iammindelesss.github.io/Obryth/` (confirmation and password-reset emails use this; the old address no longer loads). Storage buckets and the key do not change. I could not open your live site to confirm the new address loads, so open it once yourself.

## Repair notes (build 8)

**What was wrong**
1. **The migration could not finish (my mistake).** The previous script created a security rule that mentioned `download_count` and `favourite_count` before it added those columns, so it stopped with an error. That is why the comments, download-history, follows and notification tables never appeared, and why you saw "column does not exist" errors. You then added some columns and `favourites` by hand, which is why only some features came back. The new script adds each column before anything uses it.
2. **Favourites worked only half.** A hand-made `favourites` table has no counter trigger, so counts stayed at 0 and no notifications were created. The migration adds the trigger and recounts every model from the real favourite rows.
3. **Hidden error causes.** Many messages said only "Please try again" or showed raw database text. Every database error now becomes a short, safe message (for example a missing table says the owner must run the migration) and the full detail goes to the browser console.
4. **Reports table never inspected.** Its columns were unknown. The migration adapts it (adds missing columns, keeps existing rows) and removes any policy that lets browsers read or write reports directly.
5. **Broken report-email links (404).** Each email link was built from the `SITE_URL` secret exactly as typed, and a wrong value (for example the old `/filamentflow/` or `/Obrant/` addresses, which stop loading when a repository is renamed; the repository is now `Obryth`) gives a 404 on GitHub Pages. The site itself is fine: it uses `#/...` addresses, which work directly on GitHub Pages with no server rules. The function now cleans the address, loads it before sending, uses the address your website reports if `SITE_URL` fails (same website host only), and puts a loud warning in the email if neither works.

**Also confirmed by reading every database call in the code:** all column names the website uses match your real tables (`profiles`: username, display_name, bio, avatar_url, created_at; `models`: all the columns you listed). Nothing uses the old `profiles.description` column, so its text is copied into `bio` where bio is empty and the column is left alone.

## After running the migration (check it worked)

Run these in the SQL Editor. They only read.
```sql
select table_name from information_schema.tables where table_schema = 'public' order by 1;
-- expect: comments, download_events, favourites, follows, models, notification_prefs, notifications, profiles, reports
select tablename, policyname, cmd from pg_policies where schemaname in ('public', 'storage') order by 1, 2;
-- every policy on these tables should start with "orbrant", except any on storage you made yourself
select id, favourite_count, download_count from public.models order by created_at desc limit 5;
```
Send me the output if anything looks wrong.

**Old reports with broken links.** Open reports can't be recovered from old emails (only a hash of each link is stored). To make fresh links for them, run this once; it prints a working review link for each open report and invalidates the old ones:
```sql
with t as (select id, replace(gen_random_uuid()::text || gen_random_uuid()::text, '-', '') as tok from public.reports where status = 'open')
update public.reports r set token_hash = encode(sha256(convert_to(t.tok, 'UTF8')), 'hex'), attempts = 0
from t where r.id = t.id
returning r.id, r.model_title, 'https://iammindelesss.github.io/Obryth/#/remove/' || r.id || '/' || t.tok as review_link;
```

## Logo and brand assets

The uploaded file `Futuristic_OBRYTH_Emblem_on_Dark_Background.png` is a complete logo: the split-ring emblem **and** the OBRYTH wordmark in one image (1254 × 1254, opaque near-black background, 1.4 MB). It is used as supplied, so the wordmark is never added a second time.

| File in `assets/` | What it is | Used for |
| --- | --- | --- |
| `obryth-logo.webp` / `.png` (730 × 457) | The full logo, cropped to its edges, original pixels | Homepage hero |
| `obryth-logo-sm.webp` / `.png` (360 × 225) | The same crop, smaller | Navigation bar and footer |
| `obryth-emblem.webp` / `.png` (380 × 380) | The emblem only, cut from the same pixels | Studio card and the faint ring behind the closing section |
| `favicon.ico`, `favicon-32.png`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` | The emblem on the brand background | Browser tab and home-screen icons |
| `preview-calculator.webp` / `.png` | A real screenshot of the Obryth Calculator | The Calculator card on the homepage |

How the image is handled: nothing is redrawn, stretched or recoloured. The only change to the logo pixels is lowering the black point by 14 levels so the near-black background becomes true black; the page then uses `mix-blend-mode: screen`, which lets the dark background disappear into the page so only the silver artwork shows. The files are about 6–13 KB each instead of 1.4 MB. Aspect ratio is fixed by the image's own width and height, so it cannot distort. There is no transparent version because the original has none; to make one, supply a transparent PNG and replace the files in `assets/` with the same names.

Not done: no social-sharing image (`og:image`) is set, because those need a full web address and the address has just changed. Add one later once the address is settled.

## Homepage

Hero with the logo, "Ideas into experiences." and two calls to action; a product showcase ("Software worth exploring.") with Obryth Studio, Obryth Model Library and Obryth Calculator; "Built with intention." with three principles; a closing call to action; and a footer with the logo. Facts shown are real: Studio says "Coming soon" with no download, the Library card shows the live count of published models (or says there are none), and the Calculator card uses a screenshot of the real calculator. Nothing is invented: no testimonials, statistics, release dates or downloads.

Motion (all optional): entrance animation, slow glow behind the logo, a gentle logo tilt that follows the mouse (mouse only; touch screens get a slow float), scroll reveals, card lighting that follows the pointer (and lights up on keyboard focus or tap), smooth scrolling. With "reduce motion" switched on in the system, all animation stops. With JavaScript off, or if `script.js` fails to load, all content is still visible. Colours: background #090A0C, secondary #111318, cards #191B20, silver #C5C9D0, text #E7E8EB.

Navigation: Products and About scroll to their homepage sections (`#/products`, `#/about`). Settings moved to the account menu and the footer. There are no privacy or terms pages in this project, so none are linked; add links to the footer when those pages exist.

## Things that behave differently now

- **Downloading requires a signed-in account.** This makes download counts and history real and stops anonymous scraping. Storage enforces it, not just the button.
- **Download counts** go up once per person per model per 24 hours, only after the file download has started. Clearing your history never lowers a model's count.
- **Notifications** appear for comments on your models, favourites on your models and new followers (never for your own actions; repeating an action cannot create a duplicate). They refresh on every page change and every minute; there is no realtime push.
- Uploads still go live immediately; reports are the safety net.

## Reports and removal (no pre-approval)

Uploads go live in the library straight away. Safety works afterwards: any signed-in visitor can click **Report this model**. That sends you an email with a button. Clicking it opens a page on your site that shows the report; you type your password and click **Remove model permanently** (or **Keep model**). The model row and its files are then deleted.

**Be aware:** with no review, anything uploaded is public immediately, including the file download. Reports are your only safety net, so check your email. You can always remove a model yourself in Supabase (Table Editor → models → delete the row, then delete its files in Storage).

Sending email and checking the password must happen on a server, because those need secrets that can never be in a GitHub Pages site. The server part is one small Supabase **Edge Function**, `moderation-function.ts` (not uploaded to GitHub). Setup, once:

1. **Run the migration again.** Re-run all of `supabase-migration.sql` in the SQL Editor. It is safe to repeat; it now allows published uploads and adds the `reports` table. (Models you uploaded earlier that are still `pending` can be made live with `update public.models set status = 'published' where status = 'pending';`.)
2. **Get an email sender (free).** Sign up at resend.com using **the same email address that should receive reports**, then create an API key. Resend's free plan (3,000 emails a month, 100 a day) can only send to the address you signed up with, which is what we want. Check current limits on their site.
3. **Deploy the function.** Supabase → **Edge Functions → Deploy a new function → Via Editor**. Name it exactly `moderation`, paste the whole of `moderation-function.ts`, and deploy. Then open the function's settings and **turn off "Verify JWT"**. (Visitors who are not signed in must be able to open your email link; the function checks the report token, your password and the user's sign-in itself.)
4. **Add the secrets.** Edge Functions → **Secrets**, add these four:
   - `RESEND_API_KEY`: the key from step 2.
   - `ADMIN_REMOVE_PASSWORD`: a long password only you know. Never put it in any site file.
   - `SITE_URL`: your live address, e.g. `https://iammindelesss.github.io/Obryth/`. Open it in a browser first: it must load your site, not a 404. The function tidies typos (quotes, spaces, a missing slash, `#/`, `index.html`) and **checks the address loads every time before it sends an email**.
   - `OWNER_EMAIL`: where reports go (you asked for `iammindeless@gmail.com`; check the spelling matches your Resend sign-up email).
5. **Upload the 4 site files** to GitHub (build 6 in the console).
6. **Test it**: with a second account, report a model. You should get an email (check spam). Open its link, try a wrong password (refused), then the right one (model removed, and gone from the library and Storage).

How it is protected: the link holds a random 64-character token, and only a hash of it is stored. Removing needs the token **and** your password. After 8 wrong passwords a link locks (you can still remove the model in Supabase). Each person can report a model once and send at most 10 reports a day. If the email can't be sent (for example a wrong key), the report is still saved in the `reports` table, but its link cannot be recovered.

## How downloads and privacy work

- Model files live in a **private** bucket. A signed-in visitor can download a file only when a **published, free** listing points at it; the site asks Supabase for a link that expires after 60 seconds.
- Premium files can be read only by their owner. Real premium sales need a payment provider and a server-side check, which are not built.
- Preview images and profile pictures are public. Public profile data is only username, display name, bio, picture and join date, never email addresses.
- Creators can change or delete only their own rows and files, and can edit only title, description, category, tags and thumbnail; status, price, counters and owner are locked at the database level. Comments can be edited only by their author and deleted by the author or the model's owner. Notifications, favourites, download history and preferences are private to each user. Rules are enforced by the database, not the buttons.

## Test checklist (do these on the live site, with two accounts)

1. Sign in; User Profile: save a username, display name and bio; refresh; they stay.
2. Account A uploads a small STL with an image. Check the files in **Storage** and the `published` row in **Table Editor → models**.
3. Account B opens the model: comment, favourite it, download it. Check the download count goes 0 → 1, downloading again does not raise it, and Download history on B's dashboard lists it.
4. Account A's bell shows the comment and favourite. Mark one read, then mark all read. Turn off "Comments" in A's Settings, comment again as B, and check no new notification appears.
5. B follows A from A's public profile (`#/u/username`); A gets a "started following you" notification. Find A with **Find creators**.
6. B edits and deletes their own comment; B cannot edit A's comment. A can delete B's comment on A's model.
7. Signed out: cannot comment, favourite, follow or download (each asks you to sign in).
8. Premium listing: shows a price, has no working purchase or download.
9. Report a model and follow the email link (see Reports and removal).
10. Library: search by title, tag and creator name; try the file type, free/premium and sort controls.

## Test report for this version

- **Server function:** 41 logic checks against a fake database and a fake email service (not real Supabase or Resend), all passing. They cover report limits and permissions, token hashing, wrong token/password refusal, the 8-attempt lock, removal deleting the row and both files, and the email links: the link is `SITE_URL` + `#/remove/<report id>/<token>` with the real id; messy `SITE_URL` values (no slash, quotes, `#/`, `index.html`, no `https://`) are cleaned; an old 404 address falls back to the address the site reported; a different host is never used; an unusable address produces a warning in the email instead of a silent dead link; and the id and token in an email open exactly that report (changing one character is refused).
- **Browser checks against a stand-in for Supabase (not the real service), in headless Chromium:** 187 checks passing. The 129 earlier checks (all features, error messages, email-style links) were re-run on the redesigned site. 58 new homepage checks cover: the logo appears in the navigation, hero and footer, every logo decodes, proportions are preserved everywhere, the hero logo is the largest, alt text, the logo links home, no duplicated wordmark; the shipped logo matches the uploaded artwork on the logo area (average difference 4.4 of 255 brightness levels, at most 12, exactly the documented black-point lowering with no colour cast); favicon files exist and the `.ico` holds three sizes; all required copy; every link on the homepage, navigation and footer is a real route and opens the right page; Products and About scroll to their sections; the Studio card promises no download; the Library card shows the real count or says there are no models; no invented statistics; signed-in and signed-out homepage states; logo tilt (limited to 8 degrees) and its return to rest; card lighting by mouse and by keyboard Tab with a visible focus ring; scroll reveals; content visible with JavaScript off and if `script.js` fails; reduced motion; text contrast (all above WCAG AA); no sideways scrolling at 1920, 1440, 1024, 820, 390 and 360 px; mobile menu; and no old brand names on 13 pages. Found and fixed during testing: product images loading late pushed the About section down after scrolling to it (fixed by giving the card previews a fixed height).
- **Not tested by me, and the reason:** I could not open your live site, so I could not click a real email link, confirm what `SITE_URL` holds, or see the redesigned homepage as deployed. I only tested in Chromium (no Safari, Firefox or a real phone), and I did not run a build step because the project has none. I could only confirm that the old `.../filamentflow/` address now fails. I also could not run the SQL (no database available, no syntax checker), so the migration, the triggers, the security rules, real uploads, downloads, favourites, comments, notifications and the real Resend email are all unverified until you run the steps above and the checklist.
- **Not built:** replies to comments, reporting individual comments or profiles, editing a model in the site, recently viewed, Recently Added and Popular strips, follower lists, a Following feed, calculator presets, account deletion, realtime notifications, 3D preview, payments.

## Calculator formulas

Cost per gram = spool price ÷ weight; filament = cost per gram × grams; waste = filament × allowance %; electricity = (W ÷ 1000) × hours × price per kWh; total = filament + waste + electricity + maintenance + packaging + other; selling price = total × (1 + markup ÷ 100); profit = price − total; margin = profit ÷ price × 100. A 100% markup is a 50% margin.
