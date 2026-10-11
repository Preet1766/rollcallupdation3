# Campus Registrar

A college administration portal for registering students, assigning roll numbers
automatically, and collecting and verifying admission documents.

**Live app:** https://rollcall-reusable-records.lovable.app
(Built with [Lovable](https://lovable.dev); changes pushed to `main` sync back into it.)

---

## What it does

| Area | What staff can do |
| --- | --- |
| **Sign in / create account** | Anyone can create an account, but only emails on the approved staff list can use the portal. Passwords are chosen by the person themselves. |
| **Register a student** | Check an admission number, fill in the details, and the system assigns a roll number automatically. A private document-upload link is created for the student. |
| **Roll numbers** | 8-digit numbers in the format described below. Cancelled numbers are reused, lowest first. |
| **Cancel admission** | Frees the roll number. The student record and history are kept. |
| **Documents** | Students upload their documents through a private link. Staff open, verify or reject each one. |
| **Accountability** | Every registration, cancellation and document review records the staff member's name, employee ID and account ID. |
| **Students list** | Search by name, admission number, roll number or phone. Filter by status, course, year. Sort and page through results. |
| **Print** | A registration record with the roll number and the registering staff member. |
| **Staff page** (administrators) | Approve or remove staff, set their name, employee ID and role. |
| **Settings** (administrators) | College name. |

### Roll number format

Eight digits: **13 · YY · C · SSS**

| Part | Meaning |
| --- | --- |
| `13` | Always the same |
| `YY` | Admission year, last two digits (`26` = 2026) |
| `C` | Course: `0` BCA, `1` BSc, `7` MCA |
| `SSS` | Serial number `001`–`999`, counted separately for each year and course |

Example: the first BCA student admitted in 2026 gets **13260001**.

**How the next number is chosen** (this runs inside the database, so it cannot be bypassed from the browser):

1. Work out the group from the admission year and course.
2. If any numbers in that group were freed by cancellations, give the **lowest** one.
3. Otherwise give the next serial number.
4. Two registrations at the same moment are handled one after the other, and a database rule makes it impossible for two active students to hold the same number.

A cancelled student keeps their old number in their record as history, but it is free for the next student.
Other groups are never affected. Course and admission date are locked after registration because the number is built from them.

### Specializations

Artificial Intelligence & Machine Learning · Data Science · Regular

### Documents students upload

10th report card · 12th report card · Aadhaar card · APAAR ID *(optional)* · RF-3/4 form · Character certificate · Migration certificate

PDF, JPG or PNG, up to 5 MB each. Files are stored privately and only approved staff can open them.

---

## Technology

React 19 · TypeScript · TanStack Start (routing) · Tailwind CSS 4 · shadcn/ui · Supabase (PostgreSQL, authentication, private file storage).

There is no separate backend server. The browser talks to Supabase, and **the rules live in the database**:
row-level security decides who can read what, and database functions do the registering, cancelling, editing and document review.

## Folder structure

```
src/
  routes/                 pages (sign-in, dashboard, register, students, roll numbers,
                          staff, settings, student upload page, reset password)
  components/             shared pieces (layout, documents panel, print record, ui/)
  lib/college.ts          roll number helpers, validation, document list, messages
  lib/api.ts              every call to the database
  hooks/                  signed-in staff, courses, settings
  integrations/supabase/  database client and generated types
supabase/
  migrations/             the database, in order (run oldest to newest)
  tests/                  database tests
```

---

## First-time setup (do these in order)

### 1. Apply the database updates

Run the three newest files in `supabase/migrations/` **in this order**
(Supabase CLI: `supabase db push`, or paste each into the Supabase SQL editor):

1. `20261010100000_staff_access_control.sql`: approved staff list, administrator role
2. `20261010100100_roll_format_accountability.sql`: new roll format, staff names on records
3. `20261010100200_student_documents.sql`: documents and private storage

If you use Lovable, you can ask Lovable to apply them instead.

> **Existing students are converted, not deleted.** They receive new-format roll numbers in admission order.
> The update stops with a clear message if students exist in courses that are no longer offered
> (BBA, B.Com, M.Com). Delete or fix those test records and run it again.

### 2. Create your administrator login **before** anything else

The first administrator is `lovepreet01706@gmail.com` (already on the approved list by the migration).

In Supabase: **Authentication → Users → Add user → Create new user**.
Enter that email, set a long password, and tick **Auto Confirm User**.
If you already created this account on the portal earlier, you can skip this step.

> Why first: the approved list trusts the email address. Creating your account yourself
> means nobody else can register that address before you.

### 3. Check the email settings

Supabase: **Authentication → Sign In / Providers → Email**. Keep **Confirm email** switched **on**.

The portal has a **Create account** tab, so anyone can sign up, but an unapproved account sees nothing (access comes only from the approved list, and only for confirmed emails).
**Confirm email must stay on:** it proves the person really owns the address. If you ever want tighter control, you can also switch off "Allow new users to sign up" here and create logins yourself under Authentication → Users.

### 4. Allow the password-reset link

Supabase: **Authentication → URL Configuration**.
Set **Site URL** to your live address and add this to **Redirect URLs**:

```
https://rollcall-reusable-records.lovable.app/reset-password
```

(Add your own domain too if you use one, and your local address followed by `/reset-password` for local testing.)

### 5. Check for accounts you do not recognise

Before this fix, anyone could sign up and become staff. Run this in the SQL editor
and delete anyone you do not recognise under **Authentication → Users**:

```sql
select u.id, u.email, u.created_at, u.last_sign_in_at
from auth.users u
left join public.staff_allowlist a on a.email = lower(u.email)
where a.email is null
order by u.created_at;
```

Those accounts can no longer see any data (access now comes only from the approved list), but removing them is good housekeeping.

### 6. Sign in

Open the portal, sign in with your email and password, and you will see the **Staff** and **Settings** pages that normal staff do not have.
Open **Staff** and set your full name and employee ID. They appear on everything you do.

---

## Giving a staff member access

Only administrators can do this. The Staff page shows the same steps:

1. The person opens the sign-in page, chooses **Create account**, and confirms their email.
2. **Staff page →** add the same email, full name, employee ID and role.
3. They sign in. Until you add their email they can sign in but see nothing.

(Alternative: create the login yourself under Supabase → Authentication → Users → Add user with **Auto Confirm User**, then ask them to use **"Forgot your password, or setting it for the first time?"** on the sign-in page.)

**To remove someone:** Staff page → Remove. They are locked out immediately. You can also delete their login in Supabase.

> Supabase's built-in email sending has a low hourly limit. If many staff reset passwords at once, some emails may be delayed. For heavier use, connect your own email service in Supabase.

## How student documents work

1. After registering a student, the confirmation screen shows a **private upload link**. Copy it and send it to the student (it is also on their profile).
2. The student opens the link, uploads each document, and sees its status. The link shows nothing else in the portal.
3. On the student's profile, staff **View**, **Verify** or **Reject** each document (a reason is required to reject). The student sees the reason and can upload again.
4. Staff can also upload on a student's behalf.
5. If a link was shared with the wrong person, use **New link**. The old one stops working immediately.

The Students list shows how many required documents are verified for each student. Verification does not block the roll number.

---

## Common changes

**Change the college name:** Settings page (administrators).

**Add a course:** run this in the Supabase SQL editor (pick a digit from 0–9 that is not used yet; BCA 0, BSc 1, MCA 7 are taken):

```sql
insert into public.courses (name, code, sort_order) values ('BBA', 2, 4);
```

It appears in the course lists after you refresh the page.

**Change the college logo:** the logo is the graduation-cap icon in `src/components/app-shell.tsx` and `src/routes/auth.tsx`. Replace the `<GraduationCap … />` with an `<img>` pointing at a file in `public/`.

**Change the specializations:** edit `SPECIALIZATIONS` in `src/lib/college.ts` *and* the list in the `students_specialization_check` rule and `check_student_fields` function in the database (add a new migration).

---

## Running locally

```sh
cp .env.example .env     # then fill in your Supabase URL and public key
bun install              # or: npm install
bun run dev              # or: npm run dev
```

## Testing

**App logic** (roll number rules, validation, document rules). Install the test runner once, then run:

```sh
bun add -d vitest
bun run test
```

**Database rules** (allocation, cancellation and reuse, duplicates, permissions, locked fields, unapproved accounts):
paste `supabase/tests/roll_allocation_and_access.test.sql` into the Supabase SQL editor and run it.
It ends with `ALL DATABASE TESTS PASSED`, or stops with `TEST FAILED: …` saying what went wrong.
It uses far-future admission years and rolls everything back, so no data is left behind.

## Security notes

* Anyone can create an account, but the database gives nothing to an account that is not on the approved list. Keep "Confirm email" switched on.
* The public key in `.env` is meant to be visible. The database refuses everything an unapproved account tries.
* Staff cannot edit roll numbers, status or history directly. Only the database functions can change them.
* Phone, email, semester and specialization are validated in the database as well as in the browser.
* Documents live in a private storage bucket and are opened through links that expire after five minutes.
* Never commit a `service_role` key or database password.
* The student upload link is a secret. Anyone who has it can upload documents for that student.

## Deployment

With Lovable, publishing from the Lovable editor deploys the app. Apply the database updates (above) to the same Supabase project first.
