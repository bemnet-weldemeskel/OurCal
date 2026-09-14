# OurCal

A group-focused web calendar for planning shared events — built as a three-phase project for SWE 5063 (Foundations of Database and Web Development Technologies) at Kennesaw State University.

Group plans usually scatter across text threads: someone changes the date, half the group never finds out, and nobody has one place to look. OurCal pulls event creation, group membership, and invitations into a single application backed by a normalized relational database.

---

## Stack

**Backend** PHP 8 · MySQLi
**Database** MySQL 8 (MariaDB locally)
**Frontend** HTML · CSS · JavaScript · Bootstrap 4.6.1
**Local environment** XAMPP (Apache + MariaDB + PHP)
**Cloud** AWS EC2 (Ubuntu, Apache, PHP) + AWS RDS (MySQL)

---

## Features

**Dashboard** — live counts of events, groups, pending invites, and reminders, with tables for upcoming events and outstanding requests.

**Events** — create, edit, delete, and search personal and shared events. Events carry a category, a location, tags, and optional reminders.

**Groups** — three-tier permission model. Members can leave; Moderators can invite and remove; GroupAdmins can also change roles.

**Invites** — two invite types (event request and group-join request), each acceptable, declinable, or deletable, with sender and receiver tracking.

**Profile and settings** — display name, bio, status message, profile picture, and a background theme colour persisted to the database and applied across every page.

**Admin view** — platform-level accounts see aggregate statistics instead of a personal dashboard, with user-action controls hidden.

---

## Database design

The schema began as an EER model and was translated into a relational schema of **19 tables**, designed to 3NF.

### Entity types

**Strong entities** — `User`, `Group`, `Event`, `Invite`, `Category`, `Location`

**Weak entities** — `Profile`, `AccountSetting`, `Reminder`, `Availability`. Each depends on a parent for identity and uses `ON DELETE CASCADE`, so deleting a user removes their profile, settings, and availability automatically.

**Associative entity** — `Membership` resolves the many-to-many between `User` and `Group`, and carries the role a user holds in that specific group.

**Multivalued attributes** — `UserPhone`, `UserAddress`, and `EventTags` are broken into their own tables rather than being crammed into a single column.

### Subclass hierarchies

Both hierarchies are disjoint and total.

```
User  ─┬─ StandardUser   (StorageLimit, MaxGroups, AccountStatus)
       └─ Admin          (AdminLevel, Permissions, AccessJoinedDate)

Event ─┬─ PrivateEvent   (PrivacyNotes, PersonalReminder, CategoryFilter)
       └─ SharedEvent    (AccessLevel, GroupID)
```

### Data integrity

Constraints are enforced at the database layer, not only in application code.

**ENUM columns** restrict values to fixed sets — `Membership.Role`, `Invite.Status`, `Invite.InviteType`, `User.UserType`, `StandardUser.AccountStatus`.

**A CHECK constraint** on `Invite` guarantees an invite targets an event *or* a group, never both:

```sql
CHECK (
  (InviteType = 'EventRequest'     AND EventID IS NOT NULL AND GroupID IS NULL)
  OR
  (InviteType = 'GroupJoinRequest' AND GroupID IS NOT NULL AND EventID IS NULL)
)
```

**Foreign keys with `ON DELETE CASCADE`** keep dependent rows from outliving their parents.

---

## Responsive design

Three layers, rather than one:

1. **Viewport meta tag** so mobile browsers render at true device width instead of scaling a 980px desktop page.
2. **Bootstrap's 12-column grid** for layout — `col-6` for dashboard tiles, `col-md-6` for form fields that stack on phones and sit side by side on tablets and up. Tables are wrapped in `.table-responsive` so wide tables scroll horizontally instead of breaking the layout.
3. **Custom `@media` queries** at three breakpoints (≤575px, 576–991px, ≥992px) tuning padding, heading sizes, and component dimensions.

A flexbox sticky footer keeps the footer at the bottom on sparse pages without pinning it over content on full ones.

| Element | Phone | Tablet | Desktop |
|---|---|---|---|
| Navbar | Hamburger | Compact horizontal | Full horizontal |
| Form fields | Stacked | Two columns | Two–three columns |
| Tables | Scrollable | Wider | Full width |
| Page padding | 15px | 25px | 35px |

---

## Running it locally

Requires [XAMPP](https://www.apachefriends.org/).

```bash
# 1. Place the project in XAMPP's web root
#    C:\xampp\htdocs\ourcal\

# 2. Start Apache and MariaDB from the XAMPP Control Panel

# 3. Import the schema
#    Open http://localhost/phpmyadmin → Import → select schema.sql → Go
#    This creates the OurCal database, all 19 tables, and seed data

# 4. Configure the connection
cp db_config.example.php db_config.php
#    Edit with your local credentials (default XAMPP: user "root", empty password)

# 5. Open http://localhost/ourcal/
```

---

## Cloud deployment

Deployed to AWS as a bonus phase, using the same three-tier architecture spread across two instances.

```
Browser ──HTTP──▶ EC2 (Apache + PHP) ──port 3306──▶ RDS (MySQL)
```

**RDS** — `db.t3.micro`, 20 GiB, schema imported through MySQL Workbench against the RDS endpoint.

**EC2** — `t2.micro` running Ubuntu, with Apache and PHP installed directly rather than bundled:

```bash
sudo apt update
sudo apt install apache2 php php-mysql mysql-client unzip -y
```

Code was uploaded over `scp`, extracted into `/var/www/html`, and given `www-data` ownership. The RDS security group permits MySQL traffic on port 3306 **only from the EC2 security group** — not from arbitrary IPs — so the database is not publicly reachable.

The only application change between environments was the host, user, and password in `db_config.php`.

---

## Known limitations

Written as a database course project, so some things were scoped out deliberately:

- **Mixed query styles.** Inserts and updates use prepared statements with `bind_param`. Some read queries interpolate an already-cast integer directly. Prepared statements throughout would be the correct production approach.
- **No password hashing layer documented** — authentication was not the focus of the assignment.
- **No automated tests.** Validation was manual, across seeded accounts exercising event creation, invitations, search, and permission boundaries.
- **No migration tooling.** Schema changes are applied by re-importing the SQL file.

---

## Team

Built by **Bemnet Weldemeskel**, **Subat Amin**, and **Timothy Cox** for SWE 5063, Kennesaw State University.

---

## Security note

`db_config.php` is gitignored and must never be committed. Use `db_config.example.php` as the template. No RDS endpoints, master passwords, or `.pem` key files belong in this repository.
