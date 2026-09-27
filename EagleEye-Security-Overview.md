# Eagle^Eye

This is not the Eagle^Eye source code. It is a public overview of how the system copies registration data, prints RFID wristbands, and uses those wristbands to keep track of students during the day. The notes focus on identity, access control, wristbands and fobs, and operational risk.

If you are reading this for a security interview, treat it as architecture notes, not as a product dump.

## What Eagle^Eye is

<img width="768" height="432" alt="grok-image-736fbed3-1b93-4abe-851f-8e9c7d0267ab" src="https://github.com/user-attachments/assets/258e280a-31fd-4cee-990e-eac28e3e81fd" />


Eagle^Eye is a software service that a small team built and ran starting in 2019. The client is a large summer camp in the greater Seattle area. The camp still uses Eagle^Eye. Parents drop off and pick up students on a tight schedule.

The old system was a web app built for fewer than 100 students at a time. At camp scale it lagged and timed out. The camp already printed wristbands on Zebra printers with student names and camp info. Those bands had no RFID chips. There was little or no pen-and-paper process.

Eagle^Eye had to:

- Match the right student to the right adult.
- Do it in seconds, not minutes.
- Keep a live record of where each student is during the day.
- Do not lose a student. Do not send a student home with the wrong person.

We track students the way a warehouse tracks packages. Each student gets a wristband with an RFID chip. Staff phones scan the band. The backend decides what that scan means: bus check, lunch check, class roster, nurse visit, or end-of-day pickup.

Eagle^Eye is a .NET system with phone apps, several full-stack sites, SQL Server storage, and a secure API-driven print factory. Peak days mean a thousand-plus students, hundreds of devices, and more than a thousand wristbands printed in a week.

## Why this is a security problem

This is not a generic app with a login screen.

The system holds PII for minors and parents: names, schedules, pickup rights, and contact data. Incident reports and nurse visits can also hold PHI. A wrong match at pickup is a safety failure. A forged credential is a custody failure. A leak of that PII or PHI is a privacy failure.

Security work was not a separate team. It was part of how data moved, how tokens were issued, how wristbands were encoded, and how scans were accepted or rejected.

## The data factory

The camp already had a registration product, UltraCamp. We did not replace that system. We copied from it.

The flow:

1. Copy registration data from the third-party registration API.
2. Run stored procedures that turn those raw tables into the tables Eagle^Eye uses.
3. Build connected tables and views so a student record can carry class, bus route, pickup location, drop-off location, and authorized adults without extra joins at print time.
4. Store that working set in SQL Server.

Stored procedures are where raw registration data becomes Eagle^Eye data. We should not assume UltraCamp already cleaned that input. An upgrade here is to add anti-injection checks at the stored procedure step, so bad strings are rejected before they land in Eagle^Eye tables.

At print time we also normalize text so foreign characters that ZPL cannot print are converted into characters the printer can handle.

## The print factory

Printing is where the software meets the physical world.

The day before a print batch, the Eagle^Eye servers build a print list. For each student they build a data transfer object. That object is not the whole database. It is the slice needed on the wristband and in the printer: id, name, class, bus route, pickup and drop-off location, and the fields required to encode the chip.

Those objects are merged into Zebra Programming Language templates. ZPL is a short printer language. The templates already know how to:

- Encode the RFID chip.
- Print the text.
- Turn black-and-white logos and images into Zebra dot instructions, as long as the detail fits 300 DPI.

<img width="832" height="832" alt="grok-image-601c96b5-c9cd-45aa-9dea-83b40db474a1" src="https://github.com/user-attachments/assets/aacb4a86-bccf-44bd-a35c-157e0446a1e7" />

The merge step is a mail-merge style substitution. Field tokens in the template are replaced with values from the DTO. The output is a raw ZPL string.

That string is stored in a server-side print queue. Local print stations, each connected to a Zebra printer, load the next job and send it to the printer. Encoding can happen ahead of time or on demand. On a good run, request to finished wristband takes seconds. On a peak week the camp queued a thousand-plus bands.

I was responsible for that pipeline: the SQL-to-DTO path, the ZPL templates, the queue, and sending jobs to the printers.

We wrote unit tests and integration tests for this pipeline. Those tests catch software regressions.

## What happens after a wristband exists

The chip on a student wristband holds the student id. That is it.

<img width="955" height="955" alt="grok-image-37d753e5-321f-425c-9186-b23634d09515" src="https://github.com/user-attachments/assets/37ac8435-b1e3-42cc-a67a-9b56cb3b7de3" />

A staff phone running the Eagle^Eye Android app reads the chip and asks the backend what to do. The same id means different things in different screens:

- Check-in at a location marks the student present there.
- Bus stop scans confirm the student is on the right route.
- Hot lunch, class venues, and field trip sites work the same way.
- Extra roster scans give a head count.
- Nurse station scans record that a student was seen.
- Checkout shows the authorized adults registered to pick that student up.

Students were on site about eight hours and passed five to eight checkpoints in a day. A scan writes presence and location state in the database. That is how staff know where a student is and which staff member last checked them.

The checkout screen is the high-stakes one. Scanning a band does not open a free-form "release this student" button. It shows the adults who are allowed to take that student home.

## Authentication and authorization

OAuth is handled by .NET libraries. Changing or redoing that path is esoteric.

After login, OAuth issues a 24-hour JWT to the user. That JWT is how SSO works. A parent can sign in on the phone app and follow a link to an Eagle^Eye website. The same JWT still works there for those 24 hours, so parents do not have to log in twice during pickup. The expiry keeps a stolen session from lasting forever.

Most Eagle^Eye API calls require that JWT to be present. That includes Eagle^Eye to Eagle^Eye calls, not just parent or staff calls from a phone or browser. This is an area where the design could be simpler. At current use levels it does not need to be redone.

Roles are group based. The important groups are parents, staff, and administrators. After login, access is role based. A parent token does not get staff screens. A staff token does not get admin settings. Resources are checked against the role, not against "the caller reached the endpoint."

Passwords on the backend are not stored in reversible form. They are hashed with a work-factor scheme. The hasher is given a target amount of work, not a cheap one-shot hash. As machines get faster, the same target runtime produces more hashing work. Offline cracking should not get cheaper just because hardware improved. That hashing lives in the cloud service layer, not on the phones.

HTML form fields are strongly typed with Entity Framework. SQL calls from those forms use token replacement rather than raw string building. That is the control against injection and against cross-site scripting on the sites parents and staff use.

## Student wristbands

Wristbands are not encrypted. Anyone with a compatible reader can read the student id off the chip.

<img width="1408" height="939" alt="grok-image-159cc5cd-2cfe-401a-9ff6-0636972885d6" src="https://github.com/user-attachments/assets/3fb79e4b-ae84-416e-9cf0-46a05756e317" />


Knowing a student id is not useful without a valid login to Eagle^Eye. The backend is what turns an id into a name, a location, or a pickup list. A band also has a short life. It is tied to the current session week. A copy of last week's band does not check in a student who is not registered this week.

A forged band can exist. Wrong week is a clear reject: the person may exist in history, but they are not registered now. A raw id with no server access also does nothing useful. It does not unlock guardian data.

Already checked in is different. The second scan is not accepted as a new arrival, but that is not an automatic security alert. Most of the time it is not a forged band. A better next step is branching logic:

- User error. Staff scanned the same student twice.
- Server or network error. This is the most likely case in the field. Remote locations and patchy wifi can make the first scan look like it failed, then succeed on a retry after the first write already landed.
- Forgery attempt. This is the least likely case.

We treated wristbands as visible identifiers, not as secrets.

## Parent fobs

Parent pickup used to mean "show a government ID and wait while staff check a list." That was slow and easy to get wrong under pressure.

We encoded RFID keychain fobs as a faster pickup credential. Parents liked them. Pickup got shorter, and the fob felt easier than digging out an ID in a parking lot.

When a fob was written, two things were stored together in the Eagle^Eye database:

- The parent account id written to the fob.
- The factory unique serial already on that fob.

A copied payload without that factory serial does not match the issuance record. That is the check against a forged fob.

The encoding worked. The fobs were still stopped after year one because the camp could not keep up with mailing and tracking thousands of physical keys. That was a distribution problem, not a failure of the credential design. It is a bittersweet lesson. Parents wanted the fobs. The nonprofit-style logistics of issuing them at scale did not hold.

## Incident report webapp

One of the full-stack sites is an incident report app over the same Eagle^Eye databases. Staff use it for injuries, discipline, and nurse visits. It uses single sign-on, role-based access, follow-up workflow, parent contact, linked incidents when more than one student is involved, and search over saved reports.

## Logging

This is not a SIEM. We did not build system-wide security audit logging. There is a system-wide log for caught exceptions. Entries are severity tagged as info, warn, or error.

## What we optimized for

A few design choices show up again and again:

- If the system is unsure at pickup or check-in, it should not release a student or create a presence record.
- Keep the credential on the wrist cheap and the decision on the server. The chip is an index. The backend is the source of truth.
- Prefer short-lived tokens over long-lived "remember this device forever" sessions.
- Write presence and location into the database when a student is scanned. A scan that does not update state is not useful during a missing-student check.

The camp still uses Eagle^Eye. We did not have a misplaced student or an unauthorized pickup on the system we ran. That is the result that matters. The rest is how we got there.

## What this repo is for

This file is the map.

Later notes will cover individual pieces in more detail: the registration copy step, the DTO and ZPL merge, the token and role model, the wristband threat model, and the fob issuance check. Those notes will stay at the level of design and reasoning. They will not include camp data, live credentials, printer vendor account details, or production source code.

Short version of my part: I was responsible for the path from registration tables to a band on a student's wrist, and I cared a lot about what happened when that band was scanned.

<img width="900" height="520" alt="grok-image-a8201447-71f0-4d81-9123-b9e2fa1b21fa" src="https://github.com/user-attachments/assets/7fdc5afb-ec8b-452d-82cd-5b8c23c190f5" />


