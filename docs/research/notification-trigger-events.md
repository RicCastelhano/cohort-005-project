# Which events in Cadence could trigger a Notification?

Research for issue #3, a child of map #1 (in-app Notifications). This file lists the candidates only. Which ones ship is decided in a separate grilling ticket.

**Source:** the codebase at local `main` @ `979e285`. That commit isn't pushed to `origin` yet, where `main` is still `b741ddd` and has no `admin.refunds.tsx`, `refunds.server.ts` or `GLOSSARY.md`. Every `file:line` below points at `979e285`. Terms follow `GLOSSARY.md`. A **Notification** is "a message Cadence shows one specific person inside the app, telling them something happened that concerns them" (`GLOSSARY.md:98`).

**Method:** I searched for every `.insert(`, `.update(` and `.delete(` in `app/routes` and `app/lib`, read each `action`, and kept the writes that touch a person other than the actor.

## Cross-cutting facts

- **No background jobs.** Every write happens inside a React Router `action`. There is no cron, queue or webhook receiver: `grep` for `cron`, `schedule` and `setInterval` finds only UI code. `scripts/seed.ts` writes data outside any request, so seeding should not fire Notifications.
- **An outbound-message hook already exists.** Refund code returns an `Email` value from the service (`app/lib/refunds.server.ts:351`). The route sends it after the DB write via `sendEmail` (`app/lib/mailer.server.ts:67`), which uses Postmark or writes to `.outbox/`. That gives Notifications a natural seam: the service returns what should be said, and the route delivers it.
- **The send happens after the commit, not inside it.** `approveRefundRequest` / `declineRefundRequest` run in `db.transaction` (`refunds.server.ts:621`, `:709`), and `sendEmail` is awaited afterwards (`app/routes/admin.refunds.tsx:130`). If the send throws, the decision is already saved and the user sees a 500. A Notification call in the same place has the same problem, which is relevant to the map's "Delivery path" question.
- **Most other writes have no transaction.** Individual Purchase (`courses.$slug.purchase.tsx:370-371`), Team Licence (`:252-254`) and Seat claim (`redeem.$code.tsx:98-110`) are separate statements.
- **Role mapping.** A Platform Admin is `users.role = "admin"` (`app/db/schema.ts:14`). An Instructor is `courses.instructorId` for that course (`schema.ts:73`). A Team Admin is the buyer of a Team Licence (`purchases.userId`), who also gets a `team_members` row with `role = admin` (`courses.$slug.purchase.tsx:210-219`). "Staff" in code means a Platform Admin or the owning Instructor (`app/lib/access.server.ts:99`).
- **"All Platform Admins" is a group, not one person.** Refund Requests are not assigned to anyone. Telling Platform Admins means sending to every `role = admin` user, or to a provider topic.

## Candidate events with a code path today

### Refunds

| # | Event | Trigger (file:line) | Actor | Recipient(s) | Data available at that moment | Gotchas |
|---|---|---|---|---|---|---|
| R1 | Refund Request raised (becomes an Open Refund Request) | `raiseRefundRequest` inserts at `app/lib/refunds.server.ts:379-388`. Called from `app/routes/courses.$slug.refund.ts:63`, and the email is sent at `:75` | Buyer (Student, or Team Admin for a Team Licence) | **Platform Admins** (all of them): there is work to review. The buyer already gets a confirmation email (`refunds.server.ts:394-407`), so an in-app confirmation is optional. The **Instructor** could arguably be told too. | `refundRequestId`, `kind` (`individual_purchase` / `team_licence`), purchase (amount, course), buyer name and email, reason, `requestedAt` | For a Team Licence, raising the request puts its Unclaimed Seats on hold (`refunds.server.ts:117`). Nobody is told, and people who hold a Seat link have no user identity until they claim it, so they can't be notified. |
| R2 | Refund Request approved | `approveRefundRequest` at `refunds.server.ts:614-685`. Called from `app/routes/admin.refunds.tsx:111`, and the email is sent at `:130` | Platform Admin | **Buyer** (Student or Team Admin). Maybe the **Instructor** too: they lose a Student and the revenue. | Refund Request id, purchase, `amountRefunded` (= `amountPaid`), `decisionNote` (optional), `resolvedByUserId`, `resolvedAt`, course title (email built at `:736-781`) | Side effects: for an Individual Purchase, the Enrollment is deleted unless the buyer also holds a Claimed Seat (`:645-666`), which makes the Student **Lapsed**. For a Team Licence, every Unclaimed Seat gets `refundedAt` (`:638-643`). The email promises the money "is on its way", but there is no payout (see Issued and Failed Refund below). |
| R3 | Refund Request declined | `declineRefundRequest` at `refunds.server.ts:692-733`. Called from `admin.refunds.tsx:118`, and the email is sent at `:130` | Platform Admin | **Buyer** | The same as R2, with a required `decisionNote` (`:701`) | A decline takes Unclaimed Seats off hold again (they become "available"). Nobody is told this. |
| R4 | Second Platform Admin decides an already-decided request | `findOpenRefundRequest` returns 409 at `refunds.server.ts:589-595` | Platform Admin | None. This is a UI error, not a Notification. | | Listed only to rule it out. |

### Purchases, Enrollments and Seats

| # | Event | Trigger (file:line) | Actor | Recipient(s) | Data available | Gotchas |
|---|---|---|---|---|---|---|
| P1 | Individual Purchase made, which creates an Enrollment | `action` at `app/routes/courses.$slug.purchase.tsx:366-373` (`createPurchase` `:180`, `enrollUser` `:145-178`) | Student (buyer) | **Instructor** of the course (new Student, new revenue). The buyer is the actor, so a welcome message to them is a self-notification. | `userId`, `courseId`, `amountPaid` (PPP-adjusted), `country`, enrollment row | `enrollUser` has a stub, `// Would send welcome email here` (`:173-175`), but the route passes `sendEmail=false` (`:371`). No transaction. |
| P2 | Team Licence bought (Seats created) | `createTeamPurchase` at `courses.$slug.purchase.tsx:245-256`, called at `:378` | Team Admin (buyer) | **Instructor** (revenue, N Seats). For the buyer it is a self-notification, or a receipt. | purchase (`amountPaid` = PPP price × quantity), team id, list of Seat codes | This creates no Enrollment, so it doesn't change the Instructor's Student count. |
| P3 | Seat claimed, which creates an Enrollment | `redeemCoupon` at `app/routes/redeem.$code.tsx:47-113` (update `:98-104`, enrollment insert `:106-110`). Action at `:185-213` | Student (claimer) | **Team Admin** (the buyer, `purchases.userId` via `coupons.purchaseId`): "one of your Seats was claimed". The **Instructor** gets a new Student. | coupon (code, `purchaseId`, `teamId`, `courseId`), claimer `userId`, `redeemedAt`, enrollment | No transaction. Seats are shared as links, not sent to a named person (`team.tsx` lists codes at `:150-160`), so the Team Admin learns *who* claimed only through this event. A Claimed Seat makes the Team Licence unrefundable (`GLOSSARY.md:59`). If a Refund Request is Open, that is a reason to tell the Team Admin. |
| P4 | Seat claim refused (on hold or refunded) | `redeem.$code.tsx:61-67` | Student | Maybe the **Team Admin** ("someone tried to claim a held Seat") | coupon, claimer | This is a failure path. It is probably noise. |
| P5 | Enrollment ended by an approved Refund Request (Student becomes Lapsed) | `refunds.server.ts:658-665` | Platform Admin | **Buyer**, which is covered by R2. **Instructor**: their roster now shows a Lapsed Student (`app/lib/enrollments.ts`, `instructor.$courseId.students.tsx`). | as R2 | This is not a separate trigger. It is a consequence of R2. |

### Lesson comments and Instructor Q&A

| # | Event | Trigger (file:line) | Actor | Recipient(s) | Data available | Gotchas |
|---|---|---|---|---|---|---|
| C1 | Top-level comment (question) posted on a lesson | `createComment` at `app/routes/courses.$slug.lessons.$lessonId.tsx:404-435` with `parentId = null`, from the action at `:898-910` | Student (or staff) | **Instructor** of the course: a new unanswered question. It already shows up in the queue at `/instructor/questions` (`instructor.questions.tsx:42-97`). Platform Admins also see that queue for every course (`:176`). | comment row (`id`, `lessonId`, `body` Markdown, `createdAt`), course via `getCourseForLesson` (`instructorId`, title, slug) | The queue skips questions by a Platform Admin or by the course's own Instructor (`instructor.questions.tsx:66-68`). A Notification should match that rule. Comment bodies are untrusted Markdown that must be rendered with `renderComment` (`schema.ts:338-339`). Include an excerpt, not raw HTML. |
| C2 | Reply posted in a lesson thread | The same `createComment` with `parentId` set (`courses.$slug.lessons.$lessonId.tsx:417-428`) | Anyone with content access | **Author of the parent comment** (`parent.userId`). Optionally, others who replied in the thread. | reply row, parent comment (author, body), lesson, course | Skip the reply when the actor is the parent author. Replies go only one level deep (`:425`). A reply to a soft-deleted question is allowed, because `createComment` doesn't check `deletedAt`, so the parent author may still be notified. |
| C3 | Instructor/Platform Admin answers a question from the Q&A queue | `instructor.questions.tsx:188-225` (`createComment` `:211`) | Instructor or Platform Admin | **Student who asked** (`question.userId`) | question, reply body, course, lesson | This is a second code path for the same thing as C2, with a duplicated `createComment` (`instructor.questions.tsx:118-157`). A Notification hook must be added to both, or the two paths merged first. |
| C4 | Comment edited | `updateComment` at `courses.$slug.lessons.$lessonId.tsx:437-457` | Author only | Probably none: only the author can edit their own comment (`:447`). | | Probably not a candidate. |
| C5 | Comment deleted by staff (moderation) | `deleteComment` at `courses.$slug.lessons.$lessonId.tsx:459-477`, staff branch at `:464` | Instructor or Platform Admin | **Comment author**, when the actor is not the author | comment, actor, `deletedAt` | This is a soft delete. Someone has to decide whether telling people about moderation is wanted. |

### Courses (Instructor / Platform Admin actions)

| # | Event | Trigger (file:line) | Actor | Recipient(s) | Data available | Gotchas |
|---|---|---|---|---|---|---|
| K1 | Course status changed by its Instructor (Draft, Published, Archived) | `updateCourseStatus` at `app/routes/instructor.$courseId.tsx:154-161`, intent at `:695-698` | Instructor (or Platform Admin) | Published: nobody specific, because "new course" is a broadcast and broadcasts are out of scope (map #1). Archived: **enrolled Students** | course row, new status | The old status isn't read before the write, so "became Published" vs "saved as Published again" can't be told apart without an extra read. Archiving fans out to every Enrollment. |
| K2 | Course status changed by a Platform Admin | `admin.courses.tsx:76-83`, intent at `:141-144` | Platform Admin | **Instructor** of the course (someone else changed their course). Enrolled Students if Archived. | `courseId`, new status | The same "no previous status" gotcha. This one has a clear single recipient. |
| K3 | Lesson content, video or title updated | `instructor.$courseId.lessons.$lessonId.tsx:207+` (`update-lesson`), `instructor.$courseId.tsx:766+` (rename), `:753` (add lesson) | Instructor | **Enrolled Students** ("new content") | lesson, course | Fan-out to every Enrollment. Edits are frequent and small, so this is noisy. Close to the feature idea "Course Announcements" (`_internal/feature-ideas.md`). |
| K4 | Lesson or module deleted | `instructor.$courseId.tsx:304-306`, `:236-240`, intents at `:840`, `:749` | Instructor | Enrolled Students (lost progress or comments) | ids only, since the row is gone after the delete | Hard delete. Any data needed must be read before the delete. |
| K5 | Quiz replaced or deleted, which **deletes every Student's quiz attempts** | `deleteQuiz` at `instructor.$courseId.lessons.$lessonId.quiz.tsx:201-219`, called on `save-quiz` (`:400-403`) and `delete-quiz` (`:425-429`) | Instructor | **Students who had attempts** (their scores vanished) | quiz id, lesson. Attempts must be read before the delete. | Saving a quiz always deletes and recreates it, so even a typo fix wipes attempts. This is a real "concerns someone else" event, but it may really be a bug to fix rather than something to announce. |
| K6 | Course price or PPP setting changed | `instructor.$courseId.tsx:700-720` | Instructor | Probably none, since there is no wishlist or follower concept. | | Not a candidate. |

### Users

| # | Event | Trigger (file:line) | Actor | Recipient(s) | Data available | Gotchas |
|---|---|---|---|---|---|---|
| U1 | Role changed by a Platform Admin (for example, a Student becomes an Instructor) | `updateUserRole` at `app/routes/admin.users.tsx:60-67`, intent at `:126-130` | Platform Admin | **The user whose role changed** | `userId`, new role | No previous role is read first. |
| U2 | Name or email edited by a Platform Admin | `admin.users.tsx:119-124` | Platform Admin | **That user** | `userId`, name, email | This is minor. |
| U3 | Course rated | `rateCourse` at `app/routes/courses.$slug.tsx:330-368`, action at `:453-481` | Student (must be enrolled) | **Instructor** | `courseId`, rating 1–5, insert vs update (re-rate) | Ratings are stars only, with no text. A re-rate is an update, so decide whether that should notify. It could be noisy. |

### Self-only events (no other person, so listed only to rule them out)

- Lesson marked complete: `courses.$slug.lessons.$lessonId.tsx:218-243`, `:947-950`.
- Quiz submitted and auto-graded: `computeResult` at `:531-638`. The result is shown inline, and no Instructor grades anything. An Instructor might want a summary, but that is analytics, not a Notification.
- Video watch events: `app/routes/api.video-tracking.ts:30`. A Student passing the **Watch Limit** concerns only themselves. It affects whether they can raise a Refund Request, but no code marks that moment.
- Signup (`signup.tsx:78`), profile edit (`settings.tsx:66+`), course created (`instructor.new.tsx:60`), category CRUD (`admin.categories.tsx`).

## Domain events with no code path yet

| Event | Where the domain describes it | Would-be recipient | Why there's no trigger |
|---|---|---|---|
| **Issued Refund** | `GLOSSARY.md:85-87` | Buyer | There is no payout. The schema stores only `amountRefunded` on the Refund Request row (`schema.ts:246`), and nothing records that money was sent. When payout lands, the trigger would most likely be a payment-provider webhook, **outside any user request**, with no logged-in actor. |
| **Failed Refund** | `GLOSSARY.md:89-90` | Buyer and Platform Admins | Same as Issued Refund. It would also arrive asynchronously. |
| **Pending Refund** | `GLOSSARY.md:81-83` | Buyer | This is a state rather than an event. It starts at R2. |
| Course completed (Enrollment finished) | `enrollments.completedAt` (`schema.ts:134`) is read by `dashboard.tsx:27` and the instructor pages | Student, maybe the Instructor | No code writes `enrollments.completedAt`. Marking the last lesson complete doesn't set it. |
| Welcome on Enrollment | `enrollUser` stub (`courses.$slug.purchase.tsx:173-175`) | Student | It is a stub, disabled by the caller. |
| Refund Request withdrawn by the buyer | Not in the glossary | Platform Admins | There is no withdraw action. |
| Seats on hold / released, as told to people holding a Seat link | ADR `docs/adr/0003-team-licence-refunds-are-all-or-nothing.md` | People holding Seat links | Seats aren't tied to a person until claimed, so there is no recipient. |
| Course announcements, coupon activity, quiz results | `_internal/feature-ideas.md` items 1 and 23 | Enrolled Students / Team Admin | These are ideas only. Announcements are also close to "broadcast", which the map rules out of scope. |

## Summary

- **Clear one-person recipient, already backed by a service function that returns an outbound message:** R2 and R3 (buyer told of the decision).
- **Clear one-person recipient, plain route code:** P3 (Team Admin told a Seat was claimed), C2 and C3 (asker told of a reply; there are two code paths), K2 (Instructor told a Platform Admin changed their course), U1 (user told their role changed).
- **Recipient is a group:** R1 (all Platform Admins), C1 (Instructor plus all Platform Admins), K1, K3, K4 and K5 (all enrolled or affected Students).
- **Recipient is the Instructor, about revenue or roster:** P1, P2, P3, P5, U3.
- **Future and asynchronous (no request, no actor):** Issued Refund and Failed Refund.
