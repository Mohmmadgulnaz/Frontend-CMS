RENEWCRED CMS + FRONTEND INTEGRATION
=====================================

A headless CMS and public site for RenewCred, built for the Frontend
Engineering assignment. An authenticated admin can create/edit pages made
of structured content blocks (headings, paragraphs, lists, tables, LaTeX
equations, and a card grid), and the public site renders that content
live from the API instead of hardcoded markup.


STACK
-----
Backend    : Express.js + SQLite (better-sqlite3)
             Matches the preferred stack (Express). SQLite instead of
             Mongo/Postgres removes the need for a separate DB container
             while keeping the same relational shape -- swapping to
             Postgres later only touches src/config/db.js and
             src/models/*.

Frontend   : Next.js 14 (App Router) + Redux Toolkit
             Matches the preferred stack. One Next.js app serves BOTH
             the public site and the /admin panel (see Assumptions).

Auth       : JWT (jsonwebtoken) + bcryptjs
             Stateless, simple to reason about, standard for a
             decoupled API.

Math       : KaTeX (react-katex)
             Renders LaTeX equations authored in the CMS.

Validation : zod
             Per-block-type schemas so malformed content blocks are
             rejected before hitting the DB.

Infra      : Docker + docker-compose
             One command to run backend + frontend together.


ARCHITECTURE
------------
renewcred-cms/
|-- docker-compose.yml
|-- backend/                 Express API (content + auth)
|   `-- src/
|       |-- config/          db.js (SQLite), seed.js
|       |-- controllers/     authController, contentController
|       |-- middleware/      JWT guard, validation (zod), error handler
|       |-- models/          pageModel, adminModel (data access)
|       |-- routes/          /api/v1/auth, /api/v1/content
|       `-- server.js
`-- web/                     Next.js app -- public site + admin panel
    |-- app/
    |   |-- page.jsx                    public: RenewCred Standards (card grid)
    |   |-- [slug]/page.jsx             public: article pages (EV, Methane, ...)
    |   `-- admin/
    |       |-- login/page.jsx          public login screen
    |       `-- (protected)/            route group behind the auth guard
    |           |-- layout.jsx          sidebar shell + redirect-if-logged-out
    |           |-- dashboard/page.jsx  page list, delete
    |           `-- pages/
    |               |-- new/page.jsx
    |               `-- [slug]/edit/page.jsx   meta form + block editor
    |-- components/
    |   |-- BlockRenderer.jsx    public-facing block -> HTML renderer
    |   |-- BlockEditor.jsx      admin block editor (add/edit/reorder/remove)
    |   |-- SiteHeader.jsx / SiteFooter.jsx
    |   `-- StoreProvider.jsx    wraps the app in the Redux <Provider>
    |-- store/
    |   |-- store.js
    |   `-- slices/authSlice.js, contentSlice.js
    `-- lib/
        |-- api.js          axios instance + JWT interceptor (used by admin)
        `-- content.js      plain fetch used by public Server Components


CONTENT MODEL
-------------
Rather than one row per page with fixed fields, a page is a title + slug
+ status plus an ORDERED ARRAY OF BLOCKS:

  { type: 'header' | 'paragraph' | 'list' | 'table' | 'equation'
          | 'card_grid',
    data: {...},
    order: number }

`data`'s shape depends on `type` (e.g. { text } for a paragraph,
{ equation, displayMode } for math, { headers, rows } for a table). This
is the same idea as the Mongoose "Mixed field" approach in the reference
doc, just expressed as a `blocks` table (page_id, type, data JSON, order)
instead of an embedded array, since SQLite doesn't have native embedded
documents. Each block type has its own zod schema in
middleware/validation.js, so the API rejects malformed block data before
it's persisted -- this is what lets the frontend BlockRenderer trust the
shape of what it receives.

Adding a new block type (e.g. an image gallery) means: add a case to
blockDataSchemas (backend), add a case in BlockRenderer.jsx (public) and
BlockEditor.jsx (admin) -- no migration needed since `data` is stored as
JSON text.


STATE MANAGEMENT: WHAT'S IN REDUX VS. LOCAL STATE
--------------------------------------------------
Redux (global):
  Admin session/JWT (authSlice), the admin page list and the page
  currently open in the editor (contentSlice). These are read from
  multiple, unrelated components (sidebar nav, route guard, dashboard
  table, editor), so they need to live above any single component.

Local component state:
  The block editor's in-progress edits (BlockEditor.jsx's `blocks` array
  lives in the edit page's useState, not Redux) and every individual form
  input. This content churns on every keystroke and only the editor
  screen cares about it -- putting it in Redux would just add
  indirection with no other consumer.

Public site:
  Intentionally has NO Redux dependency at all. Public pages are React
  Server Components that fetch() the content API directly at request
  time (lib/content.js) and render to HTML on the server. This keeps the
  public site's JS bundle small and avoids a client-side loading flash
  for content that's read-only anyway.


AUTH
----
- POST /api/v1/auth/login verifies credentials (bcrypt) and signs a JWT.
- All /api/v1/content/admin/* routes are behind an authMiddleware that
  validates the JWT and attaches req.admin.
- All /api/v1/content/public/* routes are unauthenticated, read-only,
  and only ever return pages with status: "published".
- The Next.js admin panel stores the JWT in localStorage, attaches it
  via an axios interceptor (lib/api.js), and guards /admin/* routes
  client-side via app/admin/(protected)/layout.jsx (redirects to
  /admin/login if there's no valid session). Logout just clears the
  token -- JWTs are stateless, so there's nothing to invalidate
  server-side in this scope.


ASSUMPTIONS
-----------
1. One Next.js app instead of two React apps.
   The assignment's suggested structure has separate admin-frontend and
   public-frontend apps; here both live in a single Next.js app under
   /admin/* vs. everything else, since Next.js App Router makes it easy
   to keep the two cleanly separated (a route group + layout guard)
   while sharing the design system, avoiding duplicate build tooling,
   and letting public pages use Server Components (see above). Nothing
   about the assignment required physically separate codebases, only a
   decoupled, headless architecture, which this preserves -- the public
   site consumes the same API a fully separate frontend would.

2. SQLite instead of MongoDB/Postgres.
   Chosen so the whole stack runs in Docker with zero extra services
   (no separate DB container to provision, no connection strings to get
   wrong at evaluation time). The blocks table + JSON data column mirror
   the flexible/"Mixed" schema idea described in the assignment's
   reference doc.

3. Full-array block replace on save, not per-block PATCH endpoints.
   The editor always sends the complete ordered block list for a page
   and the backend replaces all blocks for that page in one transaction.
   Simpler to reason about and sufficient for this scope; a production
   version might diff instead of replace for very large pages.

4. Client-side route guarding for /admin.
   The layout checks Redux/localStorage and redirects; there's no
   server-side middleware validating the JWT before a protected page
   even starts rendering. Documented as a known gap rather than solved,
   since it doesn't change what data is exposed (the API itself is what's
   actually protected).

5. Card grid is a first-class block type (card_grid) rather than a
   special page.
   This was added specifically to represent the "RenewCred Standards"
   landing page from the Figma (EV / Biochar / Methane / Renewable
   Energy cards) as CMS-managed content rather than hardcoding it,
   consistent with the assignment's goal.

6. Seed data reproduces the two pages visible in the provided Figma
   (renewcred-standards and ev) so the reviewer can see real content
   immediately after docker compose up.


SETUP & RUNNING
----------------

OPTION A - Docker (recommended)

  git clone <your-repo-url>
  cd renewcred-cms
  docker compose up --build

  Public site : http://localhost:3000
  Admin panel : http://localhost:3000/admin/login
  API         : http://localhost:4000/api/v1

  The backend container seeds the database automatically on first
  start.

OPTION B - Run locally without Docker

  # 1. Backend
  cd backend
  cp .env.example .env
  npm install
  npm run seed     (creates the admin user + sample pages)
  npm run dev       (http://localhost:4000)

  # 2. Frontend (separate terminal)
  cd web
  cp .env.local.example .env.local
  npm install
  npm run dev       (http://localhost:3000)


EVALUATION CREDENTIALS
-----------------------
Seeded by `npm run seed` / on first Docker start (override via env vars,
see .env.example):

  Username : admin
  Password : ChangeMe123!


SAMPLE CONTENT
--------------
Two pages are seeded so there's real content to review immediately:

  /     "RenewCred Standards" (card grid linking to EV / Biochar /
        Methane / Renewable Energy)
  /ev   an article with headings, paragraphs, a list, a LaTeX equation,
        and a table, matching the Figma's EV standards page


API REFERENCE
-------------
Method  Path                                        Auth  Description
------  ------------------------------------------  ----  -----------
POST    /api/v1/auth/login                          -     { username, password } -> { token, admin }
GET     /api/v1/auth/me                             Yes   Current admin from JWT
GET     /api/v1/content/public/pages                -     List published pages
GET     /api/v1/content/public/pages/:slug          -     Published page + blocks
GET     /api/v1/content/admin/pages                 Yes   List all pages (any status)
GET     /api/v1/content/admin/pages/:slug           Yes   Page + blocks (any status)
POST    /api/v1/content/admin/pages                 Yes   Create page: { slug, title, description, status }
PUT     /api/v1/content/admin/pages/:slug           Yes   Update page meta
PUT     /api/v1/content/admin/pages/:slug/blocks    Yes   Replace blocks: [...] for a page
DELETE  /api/v1/content/admin/pages/:slug           Yes   Delete page (cascades blocks)


KNOWN LIMITATIONS / NEXT STEPS
--------------------------------
- No image/file upload block yet (would need object storage in a real
  deployment).
- The block editor is a structured form UI per block type rather than a
  WYSIWYG rich-text editor (TipTap/Editor.js as suggested); this was a
  deliberate scope cut -- it gives the same structured-JSON output those
  libraries would, without adding their bundle size/config for this
  assignment.
- No refresh-token rotation; JWTs are long-lived (8h) for simplicity.
- No optimistic UI/undo in the block editor -- save is explicit via the
  "Save content" button.
