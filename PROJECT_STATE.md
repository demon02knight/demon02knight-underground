# demon02knight underground — Project State

Last updated: 2026-06-11
Commit: 1abda75

---

## Deployed URL

https://demon02knight.github.io/demon02knight-underground/

Pages:
- `/` — landing, auth, username setup, rooms
- `/board.html` — public message board
- `/admin.html` — admin control panel

---

## Architecture

**Stack:** HTML + CSS + Vanilla JavaScript (ES modules)
**Auth:** Firebase Authentication (email/password)
**Database:** Firestore (realtime, no backend server)
**Hosting:** GitHub Pages (static, free)
**Deploy:** `git push origin main` — Pages rebuilds automatically

No build step. No bundler. No framework. No backend server. No paid services.

All JS is loaded via `<script type="module">` inline in each HTML file.
Firebase SDK loaded from Google CDN (`https://www.gstatic.com/firebasejs/12.9.0/`).

Firebase project ID: `demon02knight-underground`

---

## Current Features

### Authentication
- Email/password sign up via Firebase Auth
- Login / logout
- Auth state persisted across sessions by Firebase
- On first login: user document created in Firestore with empty displayName

### Username System
- User picks a username after login
- Minimum 3 characters; stored lowercase
- Uniqueness enforced via `usernames` collection lookup before write
- Username locked after first save — cannot be changed without admin unlock
- Admin can unlock any user's username from the admin panel
- Username displayed in rooms chat and on board posts

### Rooms (Private Chat)
- Three rooms: `roomA`, `roomB`, `roomC`
- Selectable via dropdown on the main page
- Realtime messages via Firestore `onSnapshot`
- Messages include: displayName, text, uid, createdAt timestamp
- Room read/write requires authentication
- Admin can delete any room message
- Listener unsubscribed on room switch and on logout

### Public Board
- Publicly readable (no login required to read)
- Authenticated users can post
- Posts include: displayName, text, uid, createdAt, reactions map
- 👍 like counter (atomic, race-safe via Firestore `increment`)
- Threaded replies per post (stored as subcollection)
- Replies are publicly readable; require auth to post
- Admin can delete any board post (replies are deleted with the post by Firestore cascade? — no, subcollection docs must be deleted separately; admin delete currently only deletes the parent post)
- Reply listeners cleaned up on each board snapshot update

### Admin Panel (`/admin.html`)
- Accessible only when logged in as admin UID
- Lists all user profiles: UID, username, locked status
- Unlock button: sets `users/{uid}.locked = false`
- Username registry: shows all claimed usernames and their owner UIDs
- Delete username reservation: removes entry from `usernames` collection
- Admin room message delete: inline delete button in rooms chat
- Admin board post delete: inline delete button on board

---

## Firestore Collections

### `users/{uid}`
```
{
  displayName: string,   // chosen username
  locked: boolean        // true after first username save
}
```
Read: authenticated users (own doc only), admin (all docs)
Write: owner if not locked, admin always

### `usernames/{username}`
```
{
  uid: string   // Firebase Auth UID of the owner
}
```
Read: public (needed for uniqueness check)
Create: authenticated user, only if uid matches own UID
Delete: admin only
Document ID is the username itself — serves as the uniqueness key.

### `rooms/{roomId}/messages/{messageId}`
```
{
  displayName: string,
  text: string,
  uid: string,
  createdAt: timestamp
}
```
roomId is one of: `roomA`, `roomB`, `roomC`
Read: authenticated users only
Create: authenticated user (uid field must match caller)
Delete: admin only
Update: not permitted (messages are immutable)

### `publicBoard/{messageId}`
```
{
  displayName: string,
  text: string,
  uid: string,
  reactions: { like: number },
  createdAt: timestamp
}
```
Read: public
Create: authenticated user (uid field must match caller)
Update: authenticated user, restricted to `reactions` field only
Delete: admin only

### `publicBoard/{messageId}/replies/{replyId}`
```
{
  displayName: string,
  text: string,
  uid: string,
  createdAt: timestamp
}
```
Read: public
Create: authenticated user (uid field must match caller)
Delete: admin only
Update: not permitted

---

## Firestore Rules

Rules file: `firestore.rules` (version-controlled in this repo)
Deploy: `firebase deploy --only firestore:rules`

Key assumptions the rules encode:

1. **Admin is a single hardcoded UID** (`rHpRdUEU0NbRui3qxmetIjQESuk1`). There is no admin role in Firestore — it is a string comparison in an `isAdmin()` helper function.

2. **Username lock is server-enforced.** The `users` update rule checks `!resource.data.locked` for non-admin writes. A locked user cannot unlock themselves even via direct API calls.

3. **UID attribution is server-enforced.** All create rules require `request.resource.data.uid == request.auth.uid`. Users cannot post as someone else.

4. **Likes are the only mutable field on board posts.** The update rule uses `affectedKeys().hasOnly(["reactions"])` — no other field can be edited after posting.

5. **Room messages and replies are immutable.** No update rule exists on these collections; default deny applies.

6. **Rooms are private.** Read requires `request.auth != null`.

7. **Board and usernames are public-read.** No auth required to read posts or check username availability.

---

## Admin Model

Admin identity: single UID `rHpRdUEU0NbRui3qxmetIjQESuk1`

Admin privileges:
- Read all user profiles (others can only read their own)
- Update any user profile regardless of locked state
- Delete any user profile
- Delete any username reservation
- Delete any room message
- Delete any board post
- Access admin panel at `/admin.html`

Admin detection:
- Client-side: `user.uid === ADMIN_UID` (controls UI visibility)
- Server-side: `isAdmin()` in Firestore rules (controls actual Firestore access)

Both layers must agree. The client check is cosmetic (hides/shows buttons). The rules check is the actual security enforcement.

---

## Known Limitations (Not Yet Fixed)

- **No username format validation beyond length.** Only `length >= 3` is checked. No character allowlist (e.g., alphanumeric + underscore). A user could set a username with spaces, punctuation, or Unicode.

- **Old username reservation not cleaned up on rename.** If admin unlocks a user and they pick a new name, the old `usernames/{oldName}` document is never deleted. The old name stays reserved permanently.

- **Admin delete of board post does not delete its replies.** Firestore does not cascade-delete subcollections. Deleting `publicBoard/{id}` leaves `publicBoard/{id}/replies/*` orphaned.

- **No `<meta viewport>` tag** on any page. Site does not scale correctly on mobile.

- **No `max-width` on body** in `style.css`. Content stretches edge-to-edge on wide screens.

- **Stale subdirectory in repo.** `demon02knight-underground/index.html` and `demon02knight-underground/style.css` are old copies committed before the feature branches. They are not served by Pages and can be deleted.

- **Duplicate 24 MB SVG asset.** `Delhi.svg` is 24 MB and not referenced in any page. Bloats the repository.

- **Reply button visible to unauthenticated users.** The button renders for guests; they are only blocked when they attempt to send.

---

## Future Roadmap

Items approved for future development (not yet built):

### Admin Dashboard Enhancements
- Rename users (requires cleanup of old `usernames` reservation on rename)
- View which room a user belongs to
- Room assignment: assign users to specific rooms
- Moderation: ban/mute with timestamp
- Admin audit log (`adminLog` collection)

### Room Permissions
- Per-room access control: users only see rooms they are assigned to
- Requires schema change: `users/{uid}.allowedRooms: string[]`
- Requires rule change: room read must check membership

### Public Board Improvements
- Emoji reactions beyond 👍 (reactions map is already extensible — no schema change needed)
- Threaded reply UX improvements (replies currently functional but UI is minimal)
- Pinned announcements (new `announcements` collection, admin-only create)

### Quality / Infrastructure
- Add `<meta viewport>` to all pages
- Restore `max-width` on body
- Username character allowlist validation (client + Firestore rule)
- Input length limits (`maxlength` on inputs + Firestore rule `text.size() <= 500`)
- Delete orphaned replies when admin deletes a board post
- Remove stale `demon02knight-underground/` subdirectory
- Remove `Delhi.svg` from repo

### Not Planned
- Media uploads (no Firebase Storage)
- Backend server
- Framework migration (React, Next.js, etc.)
- Paid services
- Public registration (site remains invite-aware by design)
