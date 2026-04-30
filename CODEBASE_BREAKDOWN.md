# UniConnect Codebase Breakdown

## High-level architecture
- **Framework**: Express (server in `app.js`), EJS templating (`views/`), Tailwind-built CSS (`styles/`, `public/stylesheets/`).
- **Data access**: Supabase (Postgres + auth + storage) via `lib/supabase.js`.
- **Session/auth**: `express-session` with session user enrichment in `middleware/auth.js`.
- **Structure**:
  - **Routes**: `routes/` map URLs to controllers/handlers.
  - **Controllers**: `controllers/` build view models and orchestrate model calls.
  - **Models**: `models/` encapsulate Supabase queries.
  - **Views**: `views/*.ejs` and `views/partials/` for server-rendered UI.
  - **Lib**: `lib/` contains shared helper logic (school scoping, profile media, storage URLs, etc).

## Request flow (common path)
1. `app.js` configures middleware, session, static assets, and the view engine.
2. `middleware/auth.attachSessionUser` enriches `req.session.auth.user` and exposes `res.locals`.
3. Route handlers call controllers and models; controllers shape view models.
4. Views render EJS templates with computed view models.

## Data model summary (schema references)
Core tables used in code paths include:
- `profiles`, `schools`
- `courses`, `course_enrollments`
- `communities`, `community_members`
- `posts`, `comments`, `reactions`
- `followers`, `post_reports`
- `admin_actions`, `user_activity_logs`

(See `db/schema.sql` for full schema context.)

---

# Feature breakdown

## 1) Authentication & session handling
**Routes**: `routes/auth.js`, `routes/index.js` (login page)

**Views**: `views/login.ejs`, `views/createAccount.ejs`, `views/ForgotPassword.ejs`

**Controllers/Handlers (heavy lifting)**
- `getRequestBody` normalizes POST fields.
- `sendFailure` decides JSON response vs redirect errors.
- `resolveProfileRole` fetches role from `profiles`.
- `router.post('/login')` authenticates with Supabase and stores session data.
- `router.post('/signup')` registers user via Supabase auth metadata.
- `router.post('/forgotpassword')` triggers Supabase password reset.
- `middleware/auth.attachSessionUser` hydrates session user profile, role, school branding, and profile media.

**Models/Data**
- Uses Supabase auth APIs plus `profiles` table lookups.

**Flow**
1. User submits login/sign-up form.
2. Handler calls Supabase auth API.
3. Session is populated (`req.session.auth`).
4. `attachSessionUser` enriches `res.locals` for templates.
5. Redirect to dashboard/feed on success.

---

## 2) Dashboard (user landing)
**Route**: `routes/dashboard.js`

**View**: `views/dashboard.ejs`

**Controllers/Handlers (heavy lifting)**
- `fetchAffiliatedCourses` and `fetchAffiliatedCommunities` load user memberships and resolve signed media URLs.
- Builds `dashboardUser` display info using `buildDisplayName` and `buildInitials`.

**Models/Data**
- Direct Supabase queries to `profiles`, `course_enrollments`, `community_members`, `courses`, `communities`.

**Flow**
1. Fetch profile and viewer school context.
2. Resolve courses/communities the user belongs to.
3. Render `dashboard` with computed cards.

---

## 3) Global Feed & Posts
**Routes**: `routes/feed.js`, `routes/posts.js`

**Views**: `views/feed.ejs`, `views/post.ejs`

**Controllers (heavy lifting)**: `controllers/postController.js`
- `buildFeedViewModel` builds user view model + feed posts.
- `buildPostsViewModel` hydrates posts with author/course/community data, likes, comments.
- `buildLikeState` and `buildCommentState` fetch likes + comment collections.
- `uploadFeedImage` uploads media to Supabase storage for posts with images.
- `togglePostLike`, `createPostComment`, `reportPost`, `deletePost` handle engagement.

**Models**: `models/postModel.js`
- `fetchGlobalFeedPosts`, `fetchActivePostById` (school-scoped visibility).
- `fetchProfilesByIds`, `fetchCoursesByIds`, `fetchCommunitiesByIds`.
- Reactions: `fetchReactionForUserAndPost`, `upsertLikeReaction`, `deleteReactionForUserAndPost`, `countLikesForPost`.
- Comments: `fetchCommentsByPostIds`, `createComment`.
- Reports: `reportPost`, `fetchPostReportByReporterAndPost`.

**Flow**
1. Feed route requests posts (`fetchGlobalFeedPosts`).
2. Controller builds a rich view model (profiles + media + scope labels + likes/comments).
3. Renders feed or returns JSON for pagination.
4. POST actions update reactions/comments or create a new post, then redirect/respond.

---

## 4) Communities
**Routes**: `routes/community.js`, `routes/community-detail.js`, `routes/create-community.js`

**Views**: `views/communities.ejs`, `views/community.ejs`, `views/create-community.ejs`, `views/manage-community.ejs`

**Controller**: `controllers/communityController.js`
- `listCommunities` builds community directory cards.
- `buildCommunityPageModel` loads a community, members, and posts.
- `buildCommunityMembersViewModel` and `buildCommunityPostsViewModel` hydrate data with profiles/media.
- `createCommunity`, `updateCommunity`, `deleteCommunity` manage lifecycle.
- `joinCommunity`, `leaveCommunity`, `createCommunityPost` handle membership + posting.

**Model**: `models/communityModel.js`
- Community CRUD: `createCommunityRecord`, `updateCommunityById`, `deleteCommunityById`.
- Memberships: `upsertCommunityMembership`, `joinCommunity`, `leaveCommunity`.
- Page data: `fetchCommunityPageData`, `resolveCommunityId` (school-scoped access).
- Cleanup: `removeCommunityWithDependencies` deletes related posts/comments/reactions/reports.

**Flow**
1. Directory loads communities + memberships.
2. Community page loads members and posts, checks school-scoped access.
3. Create/update actions validate input, write to `communities`, and log activity.
4. Join/leave and post actions update memberships and `posts`.

---

## 5) Courses
**Route**: `routes/courses.js`

**Views**: `views/courses.ejs`, `views/course.ejs`, `views/create-course.ejs`, `views/manage-course.ejs`

**Controller**: `controllers/courseController.js`
- `listCourses` builds "My Courses" and "Available" sections.
- `showCourseById` builds course view model, members, and posts.
- `createCourse`, `updateCourse`, `joinCourse`, `leaveCourse`, `createCoursePost`.
- Access checks: `buildCourseCreationAccess`, `buildCourseManageAccess`.

**Model**: `models/courseModel.js`
- Course CRUD: `fetchCourseById`, `createCourseRecord`, `updateCourseRecord`.
- Enrollments: `fetchCourseEnrollmentsForUser`, `upsertCourseEnrollment`, `deleteCourseEnrollment`.
- Posts: `fetchCoursePosts`, `createCoursePost`, `userCanPostInCourse`.

**Flow**
1. Directory loads course list based on school and enrollments.
2. Course detail loads members and posts if viewer is enrolled/instructor.
3. Create/manage flows validate inputs, write to `courses`.
4. Enroll/leave updates `course_enrollments`.

---

## 6) Profiles & Following
**Route**: `routes/profile.js`

**View**: `views/profile.ejs`

**Handlers (heavy lifting)**
- `renderProfileByUserId` builds full profile page.
- `buildProfileRenderData` combines media, school name, memberships, posts, follow state.
- `fetchFollowState` and `fetchFollowersList` build follower data.

**Models**
- `postModel` used to fetch posts + related courses/communities.
- `lib/profileMedia` for avatar/banner URLs.

**Flow**
1. Resolve canonical profile slug and access scope (school-based).
2. Fetch profile, memberships, posts, media, follower stats.
3. Render profile view or follow/unfollow via POST.

---

## 7) Settings & Profile Media
**Route**: `routes/settings.js`

**View**: `views/settings.ejs`

**Handlers (heavy lifting)**
- Loads profile and resolves avatar/banner URLs with `resolveProfileMedia`.
- `POST /settings/update` updates profile fields and media paths.

**Model/Helpers**
- `lib/profileMedia` discovers profile media columns and normalizes storage paths.
- Direct Supabase upsert to `profiles`.

**Flow**
1. Read profile + media paths.
2. Update profile fields, normalize media path, and upsert.
3. Update session and redirect to profile.

---

## 8) Admin / Moderation (Schools)
**Routes**: `routes/admin.js`, `routes/moderation.js`

**Views**: `views/schoolModeration.ejs`, plus admin templates in `views/admin*.ejs` and `views/*Moderation.ejs`.

**Handlers (heavy lifting)**
- `requireGlobalAdmin` enforces admin role.
- Create/update schools with domain normalization and optional logo URL.
- `logAdminAction` records changes in `admin_actions`.

**Model/Helpers**
- `lib/adminHelpers`, `lib/storage`, `lib/fieldUtils`.

**Flow**
1. Admin accesses moderation routes.
2. School records are listed and updated via Supabase.
3. Actions are logged for audit.

---

## 9) Storage uploads
**Route**: `routes/storage.js`

**Handlers (heavy lifting)**
- `POST /storage/signed-upload-url` creates Supabase signed upload URLs.
- Sanitizes file names and folders, verifies bucket name.

**Model/Helpers**
- Uses Supabase storage via `createSupabaseAdminClient`.

**Flow**
1. Client requests a signed upload URL.
2. Server validates input and responds with upload token/path.

---

# Shared helpers (heavy-lifting utilities)
- **`lib/schoolScope.js`**: centralizes role + school scoping and access rules.
- **`lib/profileMedia.js`**: detects schema columns, builds signed media URLs.
- **`lib/postEngagement.js`**: aggregates likes and comments for feeds.
- **`lib/storage.js`**: normalizes paths, builds signed/public storage URLs.
- **`lib/utils.js`**: name formatting, initials, profile URL slug building, date formatting.

---

# View layer summary
Key EJS templates:
- Auth: `login.ejs`, `createAccount.ejs`, `ForgotPassword.ejs`
- Feed: `feed.ejs`, `post.ejs`
- Communities: `communities.ejs`, `community.ejs`, `create-community.ejs`, `manage-community.ejs`
- Courses: `courses.ejs`, `course.ejs`, `create-course.ejs`, `manage-course.ejs`
- Profiles: `profile.ejs`
- Dashboard: `dashboard.ejs`
- Settings: `settings.ejs`
- Admin/Moderation: `schoolModeration.ejs`, `adminDashboard.ejs`, `adminUsers.ejs`, `adminUserDetail.ejs`, `reportDashboard.ejs`
- Layout: `header.ejs`, `footer.ejs`, `sidebar.ejs`, and `views/partials/*`

---

# Key flows (end-to-end examples)

## Feed creation
1. `POST /feed/posts` → `postController.createFeedPost`.
2. Optional image upload via Supabase storage.
3. Insert into `posts` table via `postModel.createFeedPostWithImage`.
4. Redirect to `/feed`.

## Community creation
1. `POST /communities/create-community` → `communityController.createCommunity`.
2. Validates input + school access.
3. Inserts community via `communityModel.createCommunityRecord`.
4. Upserts creator membership and logs user activity.
5. Redirects to new community page.

## Course creation
1. `POST /courses/create` → `courseController.createCourse`.
2. Validates length + role permissions.
3. Inserts course via `courseModel.createCourseRecord`.
4. Redirects to course detail.

## Profile follow
1. `POST /profile/:userId/follow`.
2. Ensures visibility + not following self.
3. Inserts or deletes row in `followers`.
4. Responds with JSON or redirect.

## Account creation
1. User navigates to `/createAccount` — `routes/auth.js` renders `views/createAccount.ejs`.
2. User submits the sign-up form (`POST /signup`) with `firstName`, `lastName`, `email`, `password`, `role`, and optional `schoolName`.
3. `getRequestBody` normalizes and trims all fields; missing required fields (`firstName`, `email`, `password`) return a 400 error via `sendFailure`.
4. `supabase.auth.signUp` is called with the credentials and user metadata (`first_name`, `last_name`, `role`, `school_name`) stored in Supabase auth.
5. Supabase triggers a confirmation email; the user must click the link to activate the account.
6. A database trigger (see `db/schema.sql`) automatically inserts a corresponding row into the `profiles` table, copying the metadata from auth into profile columns.
7. On success, the handler redirects to `/login` (or returns `201 { ok: true }` for JSON clients).
8. On first login, `middleware/auth.attachSessionUser` hydrates the session with the `profiles` row, resolving `role`, `schoolId`, profile media, and school branding for use across all templates.

## Administrator dashboard
1. Admin navigates to `/admin` → redirected to `/admin/dashboard` by `routes/admin.js`.
2. `requireAuth` + `requireGlobalAdmin` (`lib/adminHelpers.js`) verify the session user has `role === 'admin'`; non-admins receive a 403.
3. Four metrics are fetched in parallel:
   - **Active users today** — paginates `supabase.auth.admin.listUsers` and counts users whose `last_sign_in_at` falls on today's date in the configured timezone.
   - **Reports today** — counts rows in `post_reports` within the last 48-hour window that were created today.
   - **Accounts under penalty** — counts distinct `user_id` values in `user_penalties` that have no expiry or a future expiry.
   - **Flagged communities** — counts distinct `community_id` values in `community_reports` with `status = 'pending'`.
4. `views/adminDashboard.ejs` renders the summary cards with these four figures.

## Administrator user management
1. Admin visits `/admin/users` (optionally with `?q=<search>`) — `routes/admin.js GET /users`.
2. `fetchUsersBySearchQuery` searches `profiles` on `first_name`, `last_name`, and `email` (case-insensitive ILIKE) and returns up to 100 matches rendered in `views/adminUsers.ejs`.
3. Admin clicks a user → `GET /admin/users/:id` loads the profile, school name, engagement metrics (posts, comments, communities, courses, reports filed, reports against, active penalties), and recent activity/admin-action logs via `fetchAdminUserDetail`, `fetchUserActivityLogsPage`, and `fetchRecentAdminActionsForUser`.
4. **Update email** — `POST /admin/users/:id/update-email` validates the new address, calls `supabase.auth.admin.updateUserById` to update auth, syncs the `profiles` table, and logs a `user_email_updated` action in `admin_actions`.
5. **Penalize user** — `POST /admin/users/:id/penalize` inserts a row into `user_penalties` with an optional `expires_at`, then logs a `user_penalty_added` action in `admin_actions`.
6. All admin writes are recorded via `logAdminAction` (`lib/adminHelpers.js`) into the `admin_actions` table for audit.

## Administrator school moderation
1. Admin navigates to `/moderation` → redirected to `/moderation/school` by `routes/moderation.js`.
2. Same `requireAuth` + `requireGlobalAdmin` guards apply.
3. `GET /moderation/school` queries all rows from `schools` (ordered by name) and renders `views/schoolModeration.ejs` with name, domain, and logo URL for each school.
4. **Create school** — `POST /moderation/school` normalizes the school name (max 160 chars), strips protocol/path/`www.` from the domain, and validates it against a hostname regex. An optional logo storage path is resolved to a public URL via `buildPublicStorageUrl`. The row is inserted into `schools`; a `school_created` admin action is logged.
5. **Update school** — `POST /moderation/school/:id` performs the same validation and updates the existing row. A `school_updated` admin action is logged.
6. `fetchSchoolColumnSet` introspects `information_schema.columns` at runtime to handle schema variations (e.g., `logo_url` vs `logo` vs `logo_path`).

## Administrator report moderation
1. Admin visits `/admin/reports` — `routes/admin.js GET /reports` calls `fetchReportsPageData`.
2. Post reports, user reports, and community reports are fetched in parallel from `post_reports`, `user_reports`, and `community_reports` tables (most recent 100 each).
3. Reporter/reported labels, post/comment author IDs, and community metadata are resolved and merged into a unified sorted list rendered in `views/reportDashboard.ejs`.
4. Admin clicks a report → `GET /admin/reports/:type/:id` fetches the specific record with full context and renders `views/Report.ejs`.
5. Admin submits a decision (`POST /admin/reports/:type/:id`):
   - **Decision** must be `resolved` or `rejected`.
   - Optional **moderation actions** (`penalize_user`, `remove_content`) are applied only when decision is `resolved`:
     - `penalize_user` inserts into `user_penalties` and logs a `penalty_added_from_report` action.
     - `remove_content` soft-deletes the relevant post/comment (post report), all posts and comments by the user (user report), or all posts and comments in the community (community report) by setting `is_deleted = true`.
   - The report row's `status` and optional `admin_note` are updated in the appropriate table.
   - A `report_moderated` action is logged in `admin_actions`.
