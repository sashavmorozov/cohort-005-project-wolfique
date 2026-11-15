# Research: what learning platforms notify about

Ticket: #4 (map #1). Researched 2026-11-15.

**Question:** What do comparable learning platforms (Udemy, Coursera, Teachable, Thinkific, Maven, Canvas LMS, Kajabi) notify about in-app, for each audience (learner, instructor, platform admin, team admin)? How do they manage frequency, what is on by default, and is there evidence on what helps completion and what annoys users?

**How this was sourced:** I used the platforms' own help centres and developer docs, plus peer-reviewed or primary research. Udemy, Thinkific and Coursera help pages return 403 or fail to render when fetched directly. For those, the claims below come from the search index's extract of the named official article, not a full read of it. They are marked **(index)**. Everything else was read directly.

---

## TL;DR

1. **Most course platforms notify by email. In-app notifications are rare and mostly about community activity.** Teachable, Kajabi (outside Community), Maven, Udemy instructors and Coursera document almost only email. The platforms with a real in-app bell (Canvas, the Thinkific Community, the Kajabi Community) use it for **social and interaction events**: replies, mentions, comments, DMs, announcements and grades.
2. **The most common kinds, by audience:**
   - **Learner:** announcement posted, new content released (drip), due-date reminder, grade or feedback posted, reply or mention in a discussion, direct message, certificate or completion, and billing (payment failed, upcoming charge, subscription changes).
   - **Instructor / creator:** new enrolment (paid or free, often as separate toggles), new sale, a question or comment needing a response or moderation, submissions to grade, reviews, and course completion (often off by default).
   - **Platform admin:** subscription payment failed or cancelled, comments needing moderation, flagged or reported content, and system or administrative alerts. These are usually the same toggles as the owner's and shared across all admins.
   - **Team admin (B2B seats):** seats running low, unaccepted invitations, and requests for access. Admins also get *tools* to nudge inactive learners in bulk, rather than automatic notifications.
3. **Frequency control is mostly per-category, with a digest option.** Canvas is the reference model: each category can be set to *immediately*, *daily summary*, *weekly summary* or *off*, and each category has its own default. Udemy has a "Daily Q&A Digest" for instructors. Kajabi and Thinkific have a weekly community digest. Maven bundles reminders for items due within 24 hours of each other into one email. I found **no documented "5 new students enrolled"-style grouping in-app**, and no quiet hours, on any of the seven platforms.
4. **Defaults:** direct interactions (grades, DMs, mentions, announcements, invitations) are **on and immediate**. High-volume activity (every submission, new discussion topics, file and content changes) is **off**, or goes into a **daily or weekly summary**. Transactional and security messages, and replies to you, **cannot be turned off** (Udemy). Admin-side "FYI" events such as course completion are often **off by default** (Thinkific).
5. **Evidence:** plain reminders have **no measurable effect, or a negative one**, on MOOC completion. Commitment devices and social-norm messages do better. Batching notifications about **3 times a day** improved well-being compared with immediate delivery, and turning them off entirely made people more anxious. Stated tolerance for push is about **1 per week**: 37% of people say they would disable notifications at 2–5 per week.

---

## Per platform

### Canvas LMS (the most detailed model)

- Delivery options for each category: **Notify immediately / Daily summary / Weekly summary / Notifications off**. The API values are `immediately`, `daily`, `weekly` and `never` ([Canvas API: Notification Preferences](https://developerdocs.instructure.com/services/canvas/resources/notification_preferences)).
- Push and SMS can only be **immediate or off**. Only email gets summaries ([Instructure Community: Canvas Notifications](https://community.canvaslms.com/t5/Resource-Documents/Canvas-Notifications-PDF/ta-p/387041), via search extract).
- Instructure says that "Notify immediately" can still be held back (by up to about an hour, according to some sources) "as long as work is still happening, to prevent misleading notifications". In effect this is a debounce (same source).
- **Default frequencies.** This is the stock list, as reproduced by [Illinois State University's Canvas KB](https://help.illinoisstate.edu/spaces/Technology/pages/158892522/Manage+Your+Canvas+Notifications), which matches the Instructure resource document:

| Category | Default |
|---|---|
| Announcement, Grading, Invitation, Global Announcement | Immediately |
| Conversation message, Added to conversation, New mention, Reported reply | Immediately |
| Appointment signups / cancellations / availability, Recording ready | Immediately |
| Late grading, Submission comments, New discussion reply, Group membership update, Administrative notifications, Content link error, Blueprint sync | Daily summary |
| Due date, Grading policies | Weekly summary |
| Course content, Files, All submissions, New discussion topic, Calendar, Announcement created by you, Conversations created by me, Student appointment signups | Off |

- The audience split is implicit. Students mostly get grading, due dates, announcements and replies. Instructors get submissions, late grading and appointment signups. Admins get administrative and global notifications.

### Udemy (marketplace)

- **Instructors (index):** account-wide email notification settings, plus per-course settings that include "Lecture Ready" emails and a **Daily Q&A Digest**. "Transactional emails, Q&A reply notifications, and new direct message notifications cannot be turned off." Instructors can turn off direct messaging altogether, and students are then sent to Q&A ([Instructor Account Notifications](https://support.udemy.com/hc/articles/115008393928-Instructor-Account-Notifications)). The instructor mobile app adds push notifications for the same events.
- **Students (index):** notification preferences have two groups, "offers and promotions" and "course recommendations and instructor messages". Each course has its own toggles for **new announcements** and **promotional emails** ([Unsubscribe From Udemy Notifications](https://support.udemy.com/hc/en-us/articles/229231147-Unsubscribe-From-Udemy-Notifications)).
- **Rate limit on instructor broadcasts (index):** an instructor can send at most **4 educational announcements per course per month** ([Send Promotional Emails and Educational Announcements](https://support.udemy.com/hc/en-us/articles/229605788-Send-Promotional-Emails-and-Educational-Announcements)).
- **Learner-scheduled nudges (index):** "Learning reminders" are set up by the learner, who picks daily, weekly or once and a time. This is a separate feature from notification preferences ([How to Schedule Learning Reminders](https://support.udemy.com/hc/en-us/articles/4501093209367-How-to-Schedule-a-Learning-Calendar-Event)).
- **Udemy Business team admins (index):** an optional **low-licence reminder** at 95%, 98% or 100% of seats assigned, sent to all admins and owners **once a month** while the threshold is met ([Email Reminders for Low License Availability](https://business-support.udemy.com/hc/en-us/articles/33345372897559-Email-Reminders-for-Low-License-Availability)). An automatic invitation reminder goes to a user who hasn't accepted after **48 hours** ([Licenses & permissions section](https://business-support.udemy.com/hc/en-us/sections/4419762327959-Licenses-Permissions)). All of these are email.

### Coursera

- **Learners (index):** email preferences under Settings > Communication Preferences, with an "opt out of all Coursera emails" option ([Check and update your email communication preferences](https://www.coursera.support/s/article/learner-000001611?language=en_US)). Due-date reminders arrive by email and can be turned off per course under "Reminders". Personalised deadlines offer a **reset** after missed deadlines ([Coursera blog: flexible session-based schedules](https://blog.coursera.org/coming-soon-to-all-courses-flexible-session-based/)). Mobile push sends "tips and inspiration" to bring learners back ([Coursera blog: mobile reminders](https://blog.coursera.org/new-mobile-features-transcripts-notes-reminders/)). These blog posts are several years old.
- **Business admins:** the documentation describes **tools**, not automatic alerts. Admins can filter "invited but never joined" or "joined but not enrolled" learners and send bulk reminders ([Q1 Coursera for Business release](https://blog.coursera.org/q1-coursera-for-business-quarterly-product-release/)).

### Teachable

- **Admin notifications are all email**, toggled per user under Settings > Notifications ([Email notifications](https://support.teachable.com/en/articles/11682445-email-notifications)):
  - Enrolment (owners, authors, affiliates): new student joins the school; enrolment in a paid product; enrolment in a free product.
  - Comments (owners, authors): new comment needing moderation; new comment in your courses; reply in a thread you joined; reply to your comment.
  - Subscriptions (owners): failed payment or cancellation for non-payment; cancellation by the student.
- **Student emails the owner can turn off:** certificate of completion, comments, drip content, automatic unenrolment notification, free-product receipt, double opt-in.
- The article does not state defaults.

### Thinkific

- **The in-app bell exists only for Communities.** It sits top-right in the site header or Learner Hub, shows an unread badge up to 99, has mark read/unread, and appears only once at least one community exists **(index)** ([How Do Community Notifications Work?](https://support.thinkific.com/hc/en-us/articles/7300696848407-How-Do-Community-Notifications-Work)).
- Students can turn off community **email** but **cannot turn off web, in-app or push** notifications **(index)** ([How can students turn off email notifications?](https://support.thinkific.com/hc/en-us/articles/360039099054)).
- Admin notification settings are **shared across all Site Admins**. The Site Owner's settings apply to every admin, and Course Admins inherit them **(index)** (same article; [Course Admins](https://support.thinkific.com/hc/en-us/articles/360030735613-Course-Admins)).
- Moderators get new posts, mentions, replies and the weekly digest by email **by default**. The **Weekly Digest is off by default** and is enabled per community **(index)** ([Communities: Weekly Digest](https://support.thinkific.com/hc/en-us/articles/30998329390615-Communities-Weekly-Digest)).
- **Course completion is not sent to owners or admins by default.** The workaround is to CC yourself on the student's completion email **(index)** ([Can I get notified when my student completes a course?](https://support.thinkific.com/hc/en-us/articles/360048909674)).

### Kajabi

- **Offers:** the "notify me when a purchase is made" setting is email, enabled per offer, and accepts multiple recipients. It fires only on purchase, not when access is granted ([Manage offer notifications](https://help.kajabi.com/en/articles/17174711-manage-offer-notifications)).
- **Member emails** ([Edit automated email notifications](https://help.kajabi.com/en/articles/17174367-edit-automated-email-notifications)): announcement, assessment completion, cart abandonment, drip content released, welcome, payment action required or failed, upcoming payment, subscription cancelled, paused or resumed. Community emails cover admin posts, comments on followed posts, mentions, replies and connect requests. Affiliates get a conversion email.
- **Community (email + in-app + push)** ([Community Settings – Notifications](https://help.kajabi.com/articles/products/community/community-settings-notifications)):
  - "All notifications are enabled by default" when a member joins.
  - Admins can disable a type for the whole community. Members then see it greyed out and cannot turn it back on.
  - Members set each channel to All, Mentions or Muted.
  - Meetup reminders go out **1 day and 1 hour before**, and at the start time.
  - The **Weekly Digest** goes out Thursdays at 9am PT.
  - Some events go to admins only, such as "challenge ended" and "live room started".

### Maven (cohort courses)

- The only documented automatic student notification is a **due-date reminder email**. It is sent **the day before** the due date, **only to students who haven't completed** the item. Items due within 24 hours of each other are **bundled into one email**. It is turned on implicitly by setting a due date, and each email has an unsubscribe link ([Student notifications](https://help.maven.com/en/articles/8887252-student-notifications)).
- Everything else is instructor-authored, through campaigns, automated emails and broadcasts ([Automated course emails](https://help.maven.com/en/articles/6368763-automated-course-emails)).

---

## Patterns across platforms

| Pattern | Who does it |
|---|---|
| Per-category toggles | All |
| Per-category frequency (immediate / daily / weekly / off) | Canvas (email only) |
| Digest instead of per-event | Udemy (instructor Daily Q&A Digest), Thinkific and Kajabi (weekly community digest, opt-in or optional) |
| Bundling of near-simultaneous items | Maven (due within 24h → one email); Canvas debounce while work is in progress |
| Admin can lock a category off for everyone | Kajabi Community |
| Non-disableable kinds | Udemy (transactional, Q&A replies, DMs); Thinkific (in-app/web/push community notifications) |
| Only notify those who still need to act | Maven (only incomplete students get the reminder) |
| Rate limit on human-authored broadcasts | Udemy (4 announcements per course per month) |
| Time-based nudges | Maven (T-1 day), Kajabi meetups (T-1 day, T-1 hour, start), Udemy (learner-scheduled reminders), Coursera (deadline emails, push "come back" nudges) |
| Threshold alerts for team admins | Udemy Business (seats 95/98/100%, monthly) |

**Not found on any of the seven platforms:** in-app grouping of the form "5 new students enrolled", quiet hours or do-not-disturb, and streak notifications. Streaks are a Duolingo-style pattern and fall outside this set.

## Evidence: completion and annoyance

- **Plain reminders don't reliably help completion.** In a Stanford statistics MOOC, the reminder tool had no measurable effect on outcomes. A commitment device raised completion by about 11 percentage points, roughly 40% ([Patterson, *Can behavioral tools improve online student outcomes?*, Cornell CHERI WP165](https://archive.ilr.cornell.edu/sites/default/files/cheri_wp165_1.pdf)).
- **Deadline reminders can backfire. Social norms help.** In a randomised field experiment on a Chinese MOOC platform, a descriptive-norm message (showing peer completion rates) improved assignment completion, and a plain deadline reminder had a counterproductive effect. I saw this only through a search extract and did not read the paper itself.
- **Batching beats both constant alerts and silence.** In a randomised field experiment (n = 237), batching notifications **3 times a day** reduced stress and improved well-being compared with immediate delivery. Hourly batching made little difference, and turning notifications off entirely increased anxiety and FoMO ([Fitz et al., *Computers in Human Behavior* 101 (2019) 84–94, doi:10.1016/j.chb.2019.07.016](https://scholars.duke.edu/publication/1402953)).
- **Volume tolerance is low.** In a self-reported survey, about 10% said they would disable an app's notifications at 1 per week, 37% at 2–5 per week and 33% at 6–10 per week ([Localytics 2017 via MarketingProfs](https://www.marketingprofs.com/charts/2018/33486/how-many-mobile-push-notifications-are-too-many)). These are stated intentions from a vendor survey, so treat them as weak evidence.
- **Users want granular control over kind and timing** ([Firefox UX, *Notification Overload and User Control*](https://medium.com/firefox-ux/notification-overload-and-user-control-4b590271188e)).

**Implications for Cadence (in-app):**
- Notify immediately on direct, actionable events aimed at the user: a reply to them, a mention, a grade or feedback, an invitation, a seat assignment.
- Batch high-volume activity events, such as enrolments and new submissions for instructors. Show them as a grouped item or a digest rather than one item per event.
- Target nudges only at people who still need to act.
- Don't treat a plain "you haven't logged in" reminder as a way to drive completion.

## What I could not verify

- The **full text** of the Udemy, Thinkific and Coursera help articles. They return 403 or don't render when fetched directly, so those claims rest on the search index's extract of each official page.
- **Default on/off states** for Teachable admin notifications, Kajabi offer notifications and Udemy student and instructor categories. The docs don't state them.
- **Whether Udemy, Coursera, Teachable or Maven have an in-app notification centre** for course events at all. None is documented, apart from Udemy and Coursera mobile push.
- **The actual time of day** Canvas sends its daily and weekly summaries. It is not stated in the sources I read.
- **Coursera for Business admin-facing automatic alerts.** Only the bulk-reminder tools are documented.
- The **Chinese MOOC deadline-reminder study**. I could not trace the primary paper.
- No source gave **outcome data from the platforms themselves** on which notification kinds drive completion.
