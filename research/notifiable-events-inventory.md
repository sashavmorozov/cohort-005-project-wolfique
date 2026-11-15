# Inventory of notifiable events in Cadence

Research for issue #5 (map: #1). Question: which events that happen in Cadence today could warrant an in-app notification, and who would receive each?

**Method.** Every route `action` in `app/routes/*` was read, plus every DB write (`insert`, `update`, `delete`) in `app/routes/*` and `app/lib/*`, checked against the tables in `app/db/schema.ts`. Each row below cites the line where the write or intent branch lives, at commit `c3707a4`. No judgement is made here about which events are worth notifying.

**Roles.** `UserRole` = Student / Instructor / Admin (`app/db/schema.ts:11`). `TeamMemberRole` = Admin ("Team admin") / Member (`app/db/schema.ts:40`). "Course instructor" means the user in `courses.instructorId`. "Staff" for a course means an Admin or that course's instructor (`app/lib/access.server.ts:99`).

**Frequency** is estimated from the shape of the code (what triggers the write, and how often). There is no production traffic data in the repo.

**Vocabulary note.** The ticket points to `GLOSSARY.md`, but the repo has no such file (searched the worktree and `origin/main`). The code uses several words for one thing. The schema says *refund* (`refunds`), the learner email says *Reversal* (`app/routes/courses.$slug.refund.ts:100`), the settings loader says *chargeback* (`app/routes/settings.tsx:58`), and the course-page UI says *cancel* (`app/routes/courses.$slug.tsx:370`). Team seats are *coupons* in the schema and appear on the `/redeem/:code` route. This doc uses the schema's names.

## Event table

| # | Event | Where (file:line) | Actor | Candidate recipients by role | Rough frequency |
|---|-------|-------------------|-------|------------------------------|-----------------|
| **Comments and instructor questions** |||||
| 1 | Top-level comment (a "question") posted on a lesson | `app/routes/courses.$slug.lessons.$lessonId.tsx:898` (insert at `:431`) | Student, or staff of the course | Instructor (course instructor); Admin (questions from non-admins show in every admin's unanswered queue, `app/routes/instructor.questions.tsx:67`); other Students enrolled in the course | Medium: a few per active course per week |
| 2 | Reply posted on a lesson comment (one level deep only) | `app/routes/courses.$slug.lessons.$lessonId.tsx:898` with `parentId` (insert at `:431`) | Student or staff | Student/Instructor/Admin who wrote the parent comment; other repliers in the thread; Instructor (course instructor) when a Student replies | Medium |
| 3 | Staff reply from the Unanswered Questions queue (this answers the question and removes it from the queue) | `app/routes/instructor.questions.tsx:188` (`createComment` at `:212`, insert at `:153`) | Instructor (course instructor) or Admin | Student who asked; other participants in the thread | Medium: about one per question answered |
| 4 | Question "goes stale": no staff reply after 3 days | Computed at read time only, `app/routes/instructor.questions.tsx:160` (`STALE_AFTER_DAYS`); no write | System (time-based) | Instructor (course instructor); Admin | Low to medium; there is no event today, only a visual flag |
| 5 | Comment edited by author | `app/routes/courses.$slug.lessons.$lessonId.tsx:912` (update at `:452`) | Comment author (any role) | Participants in the thread; Instructor if it is a question | Low |
| 6 | Comment deleted (soft delete) by author **or by staff moderation** | `app/routes/courses.$slug.lessons.$lessonId.tsx:935` (update at `:472`; staff override at `:464`) | Author, or staff of the course | Comment author when staff removed it; participants in the thread | Low |
| **Enrolments, progress and quizzes** |||||
| 7 | Enrolled via individual purchase (paid or free) | `app/routes/courses.$slug.purchase.tsx:366-371` (`enrollUser` insert at `:168`; welcome-email stub at `:173`) | Student (buyer) | Student (welcome); Instructor (course instructor: new student) | Medium to high: one per purchase |
| 8 | Enrolled via team seat redemption | `app/routes/redeem.$code.tsx:163` (`redeemCoupon` at `:41`; coupon update `:84`, enrolment insert `:93`) | Student (seat holder) | Student (welcome); Team admin of the buying team (seat claimed); Instructor (course instructor) | Medium |
| 9 | Lesson started (progress set to in-progress) | `app/routes/courses.$slug.lessons.$lessonId.tsx:735`, **run from the loader** on every page view by an enrolled user (write at `:245`) | Student (implicit) | Instructor (course instructor); Team admin if the student holds a team seat | Very high: one per first view of each lesson |
| 10 | Lesson marked complete | `app/routes/courses.$slug.lessons.$lessonId.tsx:947` (write at `:217`) | Student | Student (progress or milestone); Instructor; Team admin for seat holders | High: one per lesson per student |
| 11 | Course completed | **No code path.** `enrollments.completedAt` (`app/db/schema.ts:134`) is read in `dashboard.tsx:27`, `courses.tsx:122` and `instructor.$courseId.students.tsx:37` but never written | n/a | Student; Instructor; Team admin | n/a (not implemented) |
| 12 | Quiz attempt submitted (scored, pass or fail) | `app/routes/courses.$slug.lessons.$lessonId.tsx:952` (`computeResult`; insert at `:603`, answers at `:615`; pass rule `:599`) | Student | Student (result); Instructor (course instructor); Team admin for seat holders | Medium: one per attempt, can repeat |
| 13 | Video watch event recorded (play/pause/progress pings) | `app/routes/api.video-tracking.ts:15` (insert at `:30`) | Student (implicit, from the player) | No clear recipient. Feeds refund eligibility (`courses.$slug.refund.ts:175`) and analytics | Very high: many per lesson view |
| **Purchases, refunds, team seats** |||||
| 14 | Individual purchase | `app/routes/courses.$slug.purchase.tsx:370` (insert at `:187`) | Student (buyer) | Student (receipt); Instructor (sale and revenue); Admin | Medium to high |
| 15 | Team purchase: team created if new, seats (coupons) generated | `app/routes/courses.$slug.purchase.tsx:378` (`createTeamPurchase`; team plus Team admin membership at `:214-218`, coupons at `:231`) | Any logged-in user, who becomes Team admin | Team admin (receipt and seat codes); Instructor (sale); Admin | Low to medium |
| 16 | Refund requested | `app/routes/courses.$slug.refund.ts:114` (insert at `:216`; email to learner is only a `console.log` at `:232`) | Student | Student (confirmation; the email copy promises "we will email you again when it is done"); Admin (has to review); Instructor (revenue at risk) | Low |
| 17 | Refund approved or declined | **No code path.** `RefundStatus.Approved/Declined`, `resolvedAt`, `resolvedByUserId` and `amountRefunded` exist (`app/db/schema.ts:34`, `:246-251`) but nothing writes them. The open request only shows in the learner's settings (`app/routes/settings.tsx:58`) | (would be Admin) | Student; Instructor | n/a (not implemented) |
| 18 | Team seat (coupon) claimed | Same write as #8, `app/routes/redeem.$code.tsx:84` | Student | Team admin | Medium |
| 19 | Redemption rejected (already redeemed, already enrolled, country mismatch) | `app/routes/redeem.$code.tsx:41-82` returns an error; no write | Student | Team admin (seat could not be used); Student | Low |
| **Ratings** |||||
| 20 | Course rated (first rating) | `app/routes/courses.$slug.tsx:588` (`rateCourse` insert at `:364`) | Student (enrolled) | Instructor (course instructor); Admin | Medium |
| 21 | Course rating changed | `app/routes/courses.$slug.tsx:613` (update at `:356`) | Student | Instructor | Low |
| **Course lifecycle (publish / archive / content)** |||||
| 22 | Course created (always as Draft) | `app/routes/instructor.new.tsx:112` (insert at `:60`, `CourseStatus.Draft` at `:68`) | Instructor (only; `:121`) | Admin | Low |
| 23 | Course status changed by its instructor (Draft / Published / Archived) | `app/routes/instructor.$courseId.tsx:695` (update at `:154`) | Instructor or Admin | Publish: Students (interested parties; none are tracked), Admin. Archive: enrolled Students, Team admins holding unclaimed seats | Low |
| 24 | Course status changed by Admin from Manage Courses | `app/routes/admin.courses.tsx:141` (update at `:78`) | Admin | Instructor (course instructor, when changed by someone else); enrolled Students on archive | Low |
| 25 | Course title or description edited | `app/routes/instructor.$courseId.tsx:685`, `:690` (update at `:145`) | Instructor or Admin | Instructor when Admin edited; enrolled Students (weak) | Low |
| 26 | Course price changed | `app/routes/instructor.$courseId.tsx:700` (update at `:176`) | Instructor or Admin | Admin; Instructor when Admin changed it | Low |
| 27 | PPP toggled on or off for a course | `app/routes/instructor.$courseId.tsx:716` (update at `:185`) | Instructor or Admin | Admin; enrolled Students who bought at a PPP price (access rules depend on it, `app/lib/ppp.ts:50`) | Very low |
| 28 | Sales copy edited | `app/routes/instructor.$courseId.tsx:844` (update at `:163`) | Instructor or Admin | Instructor when Admin edited | Low |
| 29 | Module added, renamed, reordered or deleted (delete also deletes its lessons) | `app/routes/instructor.$courseId.tsx:722`, `:727`, `:783`, `:740` (writes `:221`, `:229`, `:244`, `:238`) | Instructor or Admin | Enrolled Students (new or removed content); Team admins | Low to medium during authoring, low after publish |
| 30 | Lesson added, renamed, reordered, moved or deleted | `app/routes/instructor.$courseId.tsx:753`, `:766`, `:789`, `:803`, `:827` (writes `:282`, `:297`, `:310`, `:329-352`, `:305`) | Instructor or Admin | Enrolled Students ("new lesson"); Instructor when Admin edited | Low to medium |
| 31 | Lesson content, video URL, duration or repo URL edited | `app/routes/instructor.$courseId.lessons.$lessonId.tsx:207` (`updateLesson` at `:226`, update at `:86`) | Instructor or Admin | Enrolled Students (content updated) | Medium during authoring |
| 32 | Quiz saved, created or replaced (replacing **deletes all prior attempts**) | `app/routes/instructor.$courseId.lessons.$lessonId.quiz.tsx:386` (existing quiz deleted at `:402`, cascade `:200-219`) | Instructor or Admin | Students with attempts on the old quiz (results wiped); enrolled Students (new quiz) | Low |
| 33 | Quiz deleted (cascades attempts and answers) | `app/routes/instructor.$courseId.lessons.$lessonId.quiz.tsx:425` | Instructor or Admin | Students with attempts | Very low |
| **Users, roles, categories** |||||
| 34 | User signs up (always as Student) | `app/routes/signup.tsx:41` (insert at `:78`) | Visitor | Student (welcome); Admin | Medium |
| 35 | User role changed (e.g. promoted to Instructor) | `app/routes/admin.users.tsx:126` (update at `:62`) | Admin | The affected user (Student / Instructor / Admin) | Very low |
| 36 | User name or email changed by Admin | `app/routes/admin.users.tsx:119` (update at `:53`) | Admin | The affected user | Very low |
| 37 | Own profile (name or bio) updated | `app/routes/settings.tsx:90` (update at `:116`) | Any user | No clear recipient (self) | Low |
| 38 | Category created, renamed or deleted | `app/routes/admin.categories.tsx:191`, `:203`, `:215` (writes `:90`, `:110`, `:132`; delete refused while courses use it, `:125`) | Admin | Instructors with courses in a renamed category (weak); Admin | Very low |
| **Session-only (no domain state)** |||||
| 39 | Log in, log out, switch user, set dev country | `app/routes/login.tsx:40`, `app/routes/api.logout.ts:5`, `app/routes/api.switch-user.ts:11`, `app/routes/api.set-dev-country.ts:11` | Any user / developer | None (session cookie only; the last two are DevUI tooling) | n/a |

## By recipient role

Counted over rows 1-38. Several rows list more than one role.

- **Student** (as subject or audience): 1, 2, 3, 5, 6, 7, 8, 10, 11, 12, 14, 16, 17, 19, 23, 24, 25, 27, 29, 30, 31, 32, 33, 34, 35, 36. Mostly replies and answers, enrolment and welcome, quiz and progress results, refund outcomes, and content changes in courses they are enrolled in.
- **Instructor** (course instructor): 1, 2, 4, 5, 7, 8, 9, 10, 11, 12, 14, 15, 16, 17, 20, 21, 24, 25, 26, 28, 30, 35, 38. Mostly new questions, new students and sales, ratings, learner progress, and edits Admins make to their courses.
- **Admin**: 1, 4, 14, 15, 16, 20, 22, 23, 26, 27, 34, 38. Mostly the cross-course question queue, refund requests, new courses (review), sales, and signups.
- **Team admin**: 8, 9, 10, 11, 12, 15, 18, 19, 23, 29. Mostly seat claims, failed redemptions, and progress of their seat holders.

## Notable gaps and oddities

- **No obvious recipient:** video watch pings (#13), own profile edits (#37), and session actions (#39).
- **Events the data model expects but no code fires:**
  - Refund approve/decline (#17). The schema has the statuses and resolver columns, the learner email promises a follow-up, and there is no admin UI or action.
  - Course completion (#11). `enrollments.completedAt` is read in three places and never set.
- **Time-based, not action-based:** the stale-question flag (#4) is computed at render. Any notification would need a scheduler, which the repo does not have.
- **A write inside a loader:** the lesson in-progress update (#9) runs in `loader`, not `action`, so it fires on every GET by an enrolled user.
- **Placeholder notification sinks already exist.** The welcome email in `enrollUser` is an empty `if (sendEmail)` block (`courses.$slug.purchase.tsx:173`) and is called with `sendEmail=false`. The refund-request email is a `console.log` (`courses.$slug.refund.ts:232`).
- **Destructive cascades with affected users who are never told:** replacing or deleting a quiz deletes students' attempts (#32, #33), and deleting a module deletes its lessons (#29).
- **Quiz pass/fail ignores the quiz's own threshold.** `passed` is hard-coded as `score > 0.7` (`courses.$slug.lessons.$lessonId.tsx:599`), not `quizzes.passingScore`. Any "you passed" notice (#12) would inherit this.
- **Student "interest" is not tracked:** there is no wishlist or follow table, so "course published" (#23) has no defined student audience beyond already-enrolled users.
- **Team membership only ever holds the Team admin.** Seat redemption updates the coupon and enrols the user but does not add a `team_members` row (`redeem.$code.tsx:84-97`). So "Team Member" cannot be targeted as a recipient today. Seat holders can only be found through `coupons.redeemedByUserId`.
- **No `GLOSSARY.md`,** and the refund concept goes by four names (see the vocabulary note).
