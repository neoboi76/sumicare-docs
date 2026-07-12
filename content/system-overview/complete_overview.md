---
title: "complete_overview"
date: 2026-07-12
categories: ["System Overview"]
---

### Introduction

SumiCare is a web-based spa operations management platform used for managing the day-to-day operations of a spa, such as but not limited to: client appointment scheduling, task completion monitoring, reports generation, therapist and room allocation, and end-to-end transaction management. This platform can be utilized and customized by the target consumers of this product (who are predominantly enterprises operating in the wellness industry, such as spas) to suit their own particular needs. For the duration of this project, testing and implementation of SumiCare will be focused on the particular characteristics and needs of New Lasema Spa Jjimjilbang. Thus, in the creation and testing of SumiCare, the developers will cater to the needs of the aforementioned spa to shape the general platform, which can be implemented by other spas. Thus, there may be said to be a general and New Lasema Spa version of SumiCare. However, this documentation will be focused primarily on the features and structure of the general version.

### SumiCare Public Booking Website

#### Purpose

The purpose of the Public Booking Website is to allow potential clients to peruse the range of services a particular spa offers, and to book in advance reservations for massage appointments. This feature was proposed as a natural outgrowth of the discussions with the partner company (New Lasema Spa Jjimjilbang). Furthermore, besides booking and perusal, the public booking website allows clients to glean more information about the spa through the feedback of past clients, create an account on the website, and be recognized as a frequent patron by the staff of the spa, and potentially, qualify and apply vouchers for certain services as a result of the frequent patronage of a particular client to the spa. Overall, the website will serve as the public face of SumiCare, directly connected to and integrated with the internal SumiCare system, which allows, among other things, interaction with SumiCare services.

#### What we need to see

It must be explicitly stressed that this Public Booking Website is not separate from the SumiCare internal system, but is one and the same. The Public Booking Website is merely the public face; its endpoint is publicly exposed, whilst at the same time it is connected with the internal SumiCare system, connecting and interacting with the same database used by the internal system (whose endpoints are protected and not publicly exposed). For the features, what we need to be seen for the Public Booking Website is a range of features. Chief of which is the general information of a particular spa, such as New Lasema Spa Jjimjilbang: its history, its services/offerings, and social media/contact information, as well as sign-up forms, and possibly account creation for interested users. The full list of expected features is quoted from [the Q & A on January 21, 2026](01-21-26%20-%2001-21-26):

**Regarding the Public frontend website (booking system)**

* This is possible under certain conditions: clients must either fully pay (*hard reservation)* to reserve their slots, OR do a *soft reservation*, which serves to only inform Lasema that a customer is set to attend on a certain date, *without claiming the slot* (see walk-in problem)*.*"The **walk-in problem**: what if a client is late or does not attend their appointment? **Solution**: The client should be present at least 10 to 20 minutes prior to the appointment time and date. Failure to do so will deprive the client of the appointment and be given to new walk-ins. **Another solution:** Partial payment, down payment/security deposit for services. (Note: This should factor in the 15-minute period of cleaning in between massage services, i.e., if a massage is scheduled at 10:00 AM, it will, in actuality, start at 10:15 AM)."
* Clients will be identified based on their *nicknames*. If a common nickname is used (meaning that it is already taken in the system), another nickname shall be used, and the client shall also be given the option to enter their Facebook accounts for extra identification. \[Include data privacy clause and consent form when client enters the website\]. This can be implemented through account creation (email) or simple guest user creation. Emails can also be another source of unique identification.
* The data privacy clause and consent form should pop up and include accept or decline choices. The client should be made explicit that their data may be used for recommendation and application purposes.
* System must track the most frequent clients and their patterns, that is, what time of the day they usually avail of Lasema services, what day of the week, what months are most frequent, which massage services are most availed, which therapist, etc (Must explicitly ask permission to use data upon account creation).
* If clients are late for the appointment (*hard reservation*), then, say, a client enters 30 minutes late to a massage that's supposed to last one hour, then the client will only be entitled to the remaining thirty minutes (unless the client wishes to extend), but the payment will be for the full hour.
* There is a 15-minute period to clean and prepare the room. So an appointment will always begin at 15 minutes after the designated time, so, say, an 8:00 AM appointment begins at 8:15 AM, and ends at 9:15 AM.

**Regarding the Personalized Recommendation System**

* Possible, but for one major caveat: the suggestions offered by the system should not be **medical**, but merely **recreational** or for the purposes of **relaxation** (there should be a clause/disclaimer for this). La Sema will never claim to solve a medical ailment, only to induce relaxation or relief.
* The system will suggest a massage or package based on client input/profile; whether they want dry or oil, which part of the body they would like to focus on, etc., etc.
* Recommendations may contain AI.
* The recommendation system will be placed on the frontend public website and will be known to the receptionist/backend user.
* Four types of massage categories: dry, oil, hard, and soft/medium

In addition to the two aforementioned features, it shall be noted that the website, if possible, must be editable. Meaning that, Superadmins, Admins, and Managers shall be able to edit website content from the internal SumiCare system. Thus, there should be a functionality to allow a website editor to view, allowing the aforementioned users to edit the website and add their own images, text, form functionalities, etc. \[Note: This may not be possible, but try anyway\]

\[Insert screenshots\]

### Internal SumiCare System

#### Purpose

The internal SumiCare system is the core of SumiCare. And the core functionality, the core purpose of SumiCare as a whole, is to digitize previously paper-based processes for spa companies such as New Lasema Spa Jjimjilbang. As such, and as discussed in the introduction, it will be concerned with functionalities related to reports generation, client appointment and therapist scheduling, therapist management, end-to-end transaction management. Aligned with this structure, there will be five main user types for the internal SumiCare system, which are as follows:

#### Staff view & Functionalities (on hold)

The staff (therapists) view is limited to two functionalities: the tracking of their attendances (absences, tardiness, D.O., etc.) and the service callout function. The staff's view will be displayed on a screen (provided by Lasema) on which a list of names can be seen. The names of the staff members who are currently on their shifts are listed. Each display will be installed on the first floor in both the men's and women's staff areas. Once a therapist has been assigned to a client, the display will highlight the name, and a voice (AI text-to-speech) will utter a preset line, such as "\[Insert name/therapist number\] has been assigned to \[insert task\]." The idea for the voice callout came about since not all staff may be on the first floor, and may be residing on the second floor, which is the idea behind the speaker calling out the staff numbers.

For the tracking of therapist attendance, the system will include an attendance record for therapists. It will allow encoding of D.O. and recording of absences. It will also include a remarks section where therapists can provide the reason for their absence (e.g., their message). The system will function as an automated monitoring tool for attendance and HR records and will be capable of generating attendance reports.

\[Insert Receptionist View Screenshots and other pertinent material\]

#### Receptionist view & Functionalities

The receptionist has several functions associated with SumiCare. The receptionist view forms a core part of the SumiCare system since the processes that SumiCare aims to digitize are processes done manually by a receptionist. Processes such as the production of treatment slips, the assigning of therapists and rooms to clients, managing and facilitating transactions, and the collecting records thereof. The receptionist's work is directly related to the work of the manager (which will be elaborated later on.) The full list of receptionist functionalities * are found here*. Attached below are the extracted sections pertaining to receptionist functions:

1. The receptionist view is concerned with the following functions: assigning clients to rooms, assigning therapists to clients, producing a treatment slip with necessary info (the receipt shown), "skipping" a therapist, adjusting time set if in case therapist and/or client is late for their appointment, set if a therapist has been specially "requested", and request for back up therapists.

1. Concerning assigning clients to rooms: bed and/or table must be shown on the system as either "occupied" (colored, gray for male, pink for female) or "available" (white). A "common" room of, say, four beds must be of only one gender. So, if there is one male client at present, then the rest of the clients that can be admitted must be male. Exceptions are the following: if a common room is turned into a private room, a VIP room, or the 12-bed room on the third floor, where each row has three beds. For the latter, the rule applies only to each row. Thus, if there is one male client in a particular row, then the rest of the beds in that row must occupy male clients, and vice versa. Lastly, client "locker number" shall be shown on the occupied bed/table in the system *(See Q: About the client "locker number" - what does this look like? Give me a sample).*

1. Concerning assigning therapists to clients: therapists have designated "shifts" (see diagrams and Excel file). Thus, the day begins at 7:00 AM. A client schedules a massage at 7:00 AM. However, the massage will take place at 7:15 since a 15-minute period is allotted to set up the bed and/or room. This is *constant* for massage tasks that have ended. Whether there is a succeeding client or not, they must be set in order. Each "shift" is a period of time in which a particular set of therapists is able to render their services. While they are within this period (excluding those absent for various reasons or are in "skip" mode), they will be put in a "lineup" wherein they will be picked based on order *afterwards*. (Insert OS scheduling algorithms).

1. Concerning whether a therapist has been specifically "requested". To clarify, when the day begins at 7:00 AM, assuming all therapists are in, and clients come in for their appointments, the receptionist may arbitrarily from the available pool of therapists within that "shift" choose a therapist to serve a client, and can also assign available rooms based on the aforementioned conditions (see #4). However, if a client specifically requests a particular therapist, and if that therapist is available from the pool, then that therapist is directly chosen by the receptionist, and thereby the particular therapist is marked especially for that request *(See Q: Is there is special payment (in terms of commission. I know all therapists have commissions) for a therapist who is specifically requested as opposed to the ones arbitrarily chosen?)*. There are certain categories that shall be included whose function is to distinguish between a therapist who has been arbitrarily chosen and a therapist who was specifically chosen. \[^1 \]

1. Concerning skipping a therapist. A therapist can be marked as "skipped" by the receptionist upon finishing a service. The maximum allotted time for a skip is 30 minutes, a break of sorts. However, each therapist is given the option to, as it were, "skip" their "skip", or to cancel their being skipped by the receptionist, ahead of the 30-minute period.

1. Concerning adjusting the time set in case the therapist and/or client is late for their appointment. It is the receptionist's duty to adjust the time that is hardcoded into the system regarding appointment start and end times. To clarify, a massage occurs for a duration of time that is predetermined. Thus, a one-hour massage will occur for exactly one hour, and whatever the actual time elapsed or at what time the massage ended, it will be registered in the system as exactly one hour (unless the elapsed time is significant to warrant manually changing the time set in the system). Now, for instance, a client had scheduled an appointment for a massage that will begin at, say, 9:15 AM and end at 10:15 AM (assuming the massage they chose ends at exactly one hour). However, let's say the client was late for their appointment. In this scenario, it is the duty of the receptionist to adjust the start and end time of the massage that was manually encoded. The point being that the system must admit *flexibility* and manual modifications of certain "hardcoded" variables.

1. Concerning the duration of massage, there are 14 types of massage that clients can choose from, and they all have varying durations that the system must take into account (see Excel file; *See Also Q: Can a female therapist work with a male client, and vice versa?)*

1. Concerning extending a massage. A client can request an extension of a massage for an hour or more. When this happens, it is the duty of the receptionist to manually alter this. This could be manually inputted, or a button can be made available to produce the extension. An exception to this is the massage for VIP clients. VIP massage is *always* for 2 hours (1 hour in the jacuzzi and 1 hour massage).

1. The culmination of the receptionist's function is to create a *treatment slip*. That slip has various informations concerning the client and the *transaction*: their *tsn* number, name (?), locker number (?), name of the therapist they specifically requested (?), type of massage/package, time of transaction, etc. (*See Q: Could you provide the specific details regarding the treatment slip and a sample of the receipt for reference and emulation, along with the cutoff report?*). A major function of the booking system is to *digitize* this process, from stacks of papers to pure digital, exportable Excel sheets.

\[Insert Receptionist View Screenshots and other pertinent material\]

#### Manager view & Functionalities

Manager has all the functionalities of receptionists and staff, but can manage them, as well as manage reports. Both receptionists and managers can see the reports, but only the manager can see the analytics and trends shown by the report. In simpler terms, the manager has access to all reports, while the receptionist only has access to the cut off reports and day end report. They also have a comprehensive view of the transactions made encoded by the receptionist. More details are as follows:

11. Apart from producing a treatment slip per transaction, the receptionist/manager also produces a report or summary per *cutoff*, that is, per shift. So, say, after the 7:00 AM to 5 PM shift, a report is produced that contains the commissions made during that shift, massages done, the number of times a therapist was specifically requested for, etc. (see #10). The point of the system is to digitize the creation of this report, which will be used for the "end" as well as the "monthly report".

11. The culmination of each of the cutoff reports is the "end" report. Despite operating around the clock, Lasema does have an "end" of the day cutoff that usually commences *between* the end of the last shift and the beginning of the first shift for the next day (there is a space of time where there will be no shifts). Thus, this "end" report summarizes the information for each of the cutoff reports.

11. The monthly report, on the other hand, summarizes all the "end reports" made per day into a monthly all-in-one report.

In addition to said reports, managers (and perhaps receptionists) also have access to the therapist log sheet (attendance) and decking. The Therapist Decking refers to the official system for arranging the sequence or lineup of therapists, which serves as a guide for the fair and orderly assignment of clients for massage services. Therapists are organized in a "decking" or "lineup." Whenever a client arrives requesting a massage service, the therapist at the top of the decking shall be assigned first. After completing the service, the therapist shall be moved to the end of the decking, allowing the next therapist in line to be prioritized for the succeeding client. This symbol means this, while this symbol means this. Overall, the manager manages the processes receptionists and staff, and has a bird's eye view of the overall business operations of the spa via SumiCare.

\[Insert Receptionist View Screenshots and other pertinent material\]

#### Admin view & Functionalities

Admin functionality has access to all the features of Manager, Receptionist and staff. In addition to that access, admin also has access all the list of users (including other admins) and can manage them. Can update, create, and delete users below its tier. It can facilitate password resets, and add/modify permissions to each user. A user type has a default level of permissions, which can be customized by the admin. Furthermore, the admin has access to all the audit logs of the system, as well as logs per view/per action, whether that be for the manager view or the receptionist, or other admins. This is done for non-repudiation purposes. An admin cannot manage other admins.

\[Insert Receptionist View Screenshots and other pertinent material\]

#### Superadmin view & Functionalities

Superadmins has all the functionalities of all user types. In addition to that, a superadmin has the ability to manage admins. Unlike the admin, who can only view information about another admin, the superadmin can treat the admin as if it were a user, like the manager or receptionist, and can add, modify, or delete admins at will. This superadmin is the supreme admin that manages and monitors all users of the system for operational and security purposes.

\[Insert Receptionist View Screenshots and other pertinent material\]

#### User (?)

Strictly speaking, the user/client is not part of the system, but an account for a client may be created should they wish to sign up. This will be part of the system, but not a required nor important one. This entity will be used to track client data, their patterns, most requested services and therapists, as well as if they are eligible for vouchers, and their user profile for the recommendation system.

### SumiCare Business Rules & Policies

## 1. Client Identity & Privacy

* Clients are **never identified by their real names** in the system — only by their **nicknames**, including on treatment slips and all records.
* If a nickname is already taken in the system, the client must choose a different one.
* Clients may optionally link their **Facebook account** as an additional unique identifier.
* **Email** may serve as another form of unique identification (for registered accounts).
* A **data privacy clause and consent form** must be presented and accepted upon account creation or first use of the public booking website.
* Clients must explicitly consent to their data being used for recommendations, pattern analysis, and application purposes.
* The system must track **frequent client patterns**: preferred time of day, day of week, month, massage type, and preferred therapist — but only with explicit consent.

---

## 2. Room Assignment Rules

* Each bed or table must display its status as either **occupied** (colored) or **available** (white).
  * Gray = Male client occupying the space
  * Pink = Female client occupying the space
  * White = Available
* The client's **locker number** and the **therapist's nickname** must be displayed on each occupied bed or table.
* The **elapsed time** of the ongoing session must also be visible per occupied bed/table.
* **Gender segregation rule for common rooms**: A common room (e.g., four beds) must be of **one gender only**. If one male client is currently occupying a bed in a common room, all remaining beds in that room may only be assigned to male clients, and vice versa.
  * **Exceptions** to the gender rule:
    * A common room converted into a **private room**
    * A **VIP room**
    * The **12-bed room on the third floor**: the gender rule applies per **row** (three beds per row), not the entire room. Each row is gender-segregated independently.
* The system must show **therapist availability** separately for male and female therapists.

---

## 3. Therapist Lineup / Decking System

* Therapists are organized into an **ordered lineup (decking)** — a queue determining who is assigned next to an incoming client.
* Clients are assigned to the **therapist at the top of the decking** first.
* After completing a service, the therapist is **moved to the end of the decking**, allowing the next therapist in line to be prioritized.
* **Newest shift takes priority**: When a new shift opens, its therapists are placed **in front** of the lineup ahead of therapists still remaining from the previous shift. This is effectively a "latest shift first" algorithm.
* The lineup **expands** as overlapping shifts open and **contracts** as therapists finish their shifts and are removed.
* **Requested therapists** (specifically chosen by a client) are pulled from the available pool directly and do not follow the standard decking order for that transaction. However, they **retain their position** in the lineup — meaning after completing the requested service, they are still where they would have been, but with an additional service counted. This gives them more services without losing their queue position.
* **Backup therapists never appear in the automatic lineup** — they must be **manually inserted** by the receptionist when needed.
* A therapist only appears in the lineup if:
  1. It is within their designated shift, **and**
  1. They have **clocked in** via the biometrics system.

---

## 4. Shift Structure

There are **five shifts**, all operating in a 24-hour cycle:

|Shift|Time|Therapist Count|
|-----|----|---------------|
|Shift 1|7:00 AM – 5:00 PM|15|
|Shift 2|12:00 Noon – 10:00 PM|10|
|Shift 3|2:30 PM – 12:30 AM|15|
|Shift 4|5:00 PM – 3:00 AM|20|
|Shift 5|7:30 PM – 7:30 AM|20|

* Shifts are **overlapping** by design, meaning multiple shifts may be active simultaneously.
* There is a **gap of time between the last shift ending and the first shift of the next day** beginning — this is when the "end-of-day" report is generated.

---

## 5. Booking & Session Time Rules

* The **operational day begins at 7:00 AM**.
* A **15-minute buffer** is always allotted between sessions for room/bed preparation and cleaning — this is constant regardless of whether the previous session had a client immediately after.
  * Example: A 7:00 AM appointment actually begins at **7:15 AM**.
  * Example: A 10:00 AM scheduled appointment begins at **10:15 AM**.
  * This 15-minute rule must be factored into the online booking system for clients scheduling in advance.
* **Session durations are predetermined and fixed** per massage type. The system registers the transaction as the exact predetermined duration unless there is a significant deviation warranting manual adjustment.
* The receptionist has the ability to **manually adjust** start and end times in cases where a client or therapist is late. The system must support this flexibility.
* If a client is late for a **hard reservation**, they are entitled only to the **remaining time** of the session (e.g., arriving 30 minutes into a 1-hour session means only 30 minutes), but payment is still for the **full session**.

---

## 6. Massage Services Catalogue

All 14 massage types, their equipment, and durations:

|Service|Duration|Equipment|Commission (per therapist)|
|-------|--------|---------|--------------------------|
|Foot Reflex|1 hr|Bed|₱120|
|Shiatsu|1 hr|Bed|₱120|
|Swedish|1 hr|Bed|₱120|
|Deep Tissue|1 hr|Bed|₱120|
|Lomi-Lomi|1 hr|Bed|₱120|
|Lymphatic|1 hr|Bed|₱120|
|Combination|1 hr|Bed|₱120|
|Salt Glow Scrub|1 hr|Table|₱120|
|Milk Bath Scrub|1 hr|Table|₱120|
|Dae Mi DI (Korean Scrub)|30–45 min|Table|₱200 (fixed)|
|Thai Massage|1 hr 15 min|Bed|₱150 (fixed)|
|Aromatherapy w/ Reflex|1 hr 30 min|Bed|₱180 (fixed)|
|Ventosa Massage|1 hr 30 min|Bed|₱220 (fixed)|
|Tandem Massage|1 hr|Bed|₱240 (requires 2 therapists)|

* Services marked as **fixed** (Dae Mi DI, Thai, Aromatherapy, Ventosa, Tandem) have non-standard rates and durations that **cannot be extended or modified** in the same way as regular massages.
* **Four massage categories** for the recommendation system: **dry, oil, hard, and soft/medium**.

---

## 7. Session Extension Rules

* A client may request a **session extension** of 30 minutes or more (up to an additional hour or beyond).
* Extension is handled **manually by the receptionist** — either via direct input or a dedicated extension button.
* **Extension commission rate**: ₱120 ÷ 2 = **₱60 per additional 30 minutes** for the therapist.
* **VIP massage is always exactly 2 hours** (1 hour in the jacuzzi + 1 hour massage). No extension rule applies differently for VIP — the duration is fixed at 2 hours total.

---

## 8. Commission Rules

* Commission is calculated **per service rendered, per duration**.
* Standard rate: **₱120 per hour** for regular massage services.
* For services with fixed special rates (see Section 6), those fixed rates apply.
* **Tandem Massage** (requires 2 therapists): total commission is ₱240 — split between the two therapists.
* **There is no special (higher) commission rate** for a therapist who is specifically requested by a client versus one arbitrarily assigned.
* A specifically requested therapist's service is counted as **additional work** on top of their decking position — they still retain their lineup spot.
* The system must track the **number of times each therapist is specifically requested** — this data is used in monthly/annual summaries and top-10 most-requested therapist bonuses.
* Every year, management awards bonuses to the **top 10 most-requested therapists** based on the annual data.

---

## 9. Therapist Skip Rules

* A therapist can be marked as **"skipped"** by the receptionist after completing a service — this functions as a short break.
* Maximum skip duration: **30 minutes**.
* The therapist has the option to **cancel their skip early** (before the 30-minute period ends) — the system must support this.

---

## 10. Specific Therapist Request Rules

* A client may request a **specific therapist** by name/number.
* If the requested therapist is **available** in the current pool, the receptionist assigns them directly.
* The transaction must be **flagged** in the system as a "specifically requested" assignment (distinguishing it from an arbitrary assignment).
* The requested therapist is pulled from the pool without losing their lineup position.
* Reports must differentiate between **arbitrary assignments** and **specific requests** per therapist.

---

## 11. Gender Assignment Rules (Therapist–Client)

* **Male therapists** may only massage **male clients** as the default rule.
* **Female therapists** may massage both **male and female** clients.
* However, some female clients may specifically request a **male therapist** — this is permissible as an exception.

---

## 12. Backup Therapist Rules

* Backup therapists are called in when **regular shift therapists are insufficient to meet client demand**.
* Backup therapists are **not pre-scheduled** and are not the same individuals every time.
* They are paid by **commission** at the standard rates.
* Backup therapists **never automatically appear in the lineup** — they must be **manually inserted** by the receptionist.
* Their names must be **recorded in all reports** (cutoff, day-end, monthly) and clearly **distinguished** from regular staff.
* Backup therapists do **not** have a fixed staff number like regular therapists.

---

## 13. VIP Client Rules

* **VIP massage is fixed at 2 hours**: 1 hour in the jacuzzi + 1 hour massage.
* VIP clients have dedicated room tiers: **Diamond, Ruby, Emerald, and Rosegold**.
* All VIP requests are handled at the **common receptionist area** — the system does not need a separate VIP receptionist terminal.
* VIP treatment slips are distinct from regular treatment slips (separate format).

---

## 14. Treatment Slip Rules

* A treatment slip is produced **per transaction** by the receptionist.
* **Content of a treatment slip** includes:
  * TSN (Transaction Sequence Number)
  * Client nickname
  * Locker number
  * Name of specifically requested therapist (if applicable)
  * Type of massage/package
  * Time of transaction (start and end)
  * Room number
  * Therapist name/nickname
* **Process flow for treatment slip creation**:
  1. Upon client check-in, the front desk creates the treatment slip.
  1. The treatment slip is delivered to the massage receptionist.
  1. The receptionist reviews the order details and has the client sign it upon arrival.
  1. The receptionist fills in the room number and therapist name (unless a specific therapist was requested).
  1. Once the client is in the room, the therapist is called, signs the treatment slip, and enters the designated room.
* The system must **digitize** this entire process — eliminating the manual paper-based flow and replacing it with a digital, exportable system.

---

## 15. Report Hierarchy & Rules

The system must generate three tiers of reports:

### 15a. Cutoff Report (Per Shift)

Generated at the end of each shift. Contains:

* Commissions earned during that shift
* Number of massages completed
* Number of specific therapist requests
* Therapist-by-therapist breakdown
* Backup therapist records (if any)

### 15b. End-of-Day Report

Generated between the end of the last shift and the beginning of the first shift of the next day. Summarizes all cutoff reports for the full operational day.

### 15c. Monthly Report

Summarizes all end-of-day reports for the calendar month. Used for financial oversight and annual performance reviews.

### 15d. Additional Report Requirements

All reports must include:

* Most requested therapist (ranked)
* Most requested massage service
* Instances where a client requested a therapist change
* These records are used to **evaluate therapist performance**
* Reports must be **accessible remotely** (i.e., outside the company's premises)
* Reports must be **exportable** (e.g., to Excel)

---

## 16. Online Booking / Public Reservation Rules

* Two types of reservations are supported:
  * **Hard Reservation**: Client pays in full upfront to secure the slot.
  * **Soft Reservation**: Client signals intent to attend without claiming the slot. The slot is not guaranteed.
* **Walk-in policy**: Clients must be present at least **10–20 minutes before** their scheduled appointment. Failure to arrive on time forfeits the slot to walk-in clients.
* **Late arrival policy (hard reservation)**: A late client is only entitled to the **remaining session time** but is charged for the **full session**.
* **Partial payment / security deposit** may be used as an alternative mechanism to protect reserved slots.
* The system must factor in the **15-minute preparation buffer** when displaying appointment start times to clients on the booking website.
* Client accounts on the public website can be created via **email** or as a **guest user**.
* A **data privacy clause** and **consent form** must appear before account creation, with explicit Accept/Decline options.

---

## 17. Personalized Recommendation System Rules

* The recommendation engine **must not offer medical advice** — suggestions are strictly **recreational and for relaxation purposes only**.
* A **disclaimer/clause** must accompany all recommendations stating that Lasema does not claim to address medical conditions.
* The recommendation is based on client input/profile (e.g., preference for dry or oil, which body areas to focus on, etc.).
* Four massage categories are used as the basis: **dry, oil, hard, and soft/medium**.
* The recommendation system is accessible on both the **public client-facing website** and is visible to the **receptionist** on the internal backend.
* AI may be incorporated into the recommendation engine.

---

## 18. Attendance & Biometrics Rules

* Therapist attendance is tracked via a **dedicated biometrics system (fingerprint)** installed in the therapists' barracks — separate from the general employee biometrics.
* A therapist **only appears in the SumiCare lineup/decking** if:
  1. It is within their designated shift, **and**
  1. They have **clocked in** via the biometrics system.
* The general employee biometrics system covers **all employees**, not just therapists, and is separate from SumiCare's dedicated biometrics.
* The system should support **attendance records** for therapists, including:
  * Days off (D.O.)
  * Absent records
  * Remarks section for therapist-submitted reasons for absence
  * Automated attendance report generation
* D.O. and remarks should be manually inputted by the receptionist.

---

## 19. Staff View Rules

* The staff view is a **passive display** — it does not require a login account.
* Displayed on a **large screen (TV)** installed on the first floor, in both men's and women's staff areas.
* The view shows the **names of therapists currently on shift**.
* When a therapist is assigned a client, their name is **highlighted** on the display.
* A **text-to-speech voice callout** announces the assignment (e.g., "\[Therapist name/number\] has been assigned to \[task\]") — designed to reach therapists who may be on the second floor.
* Interactions with the system from the staff view are **very limited**.
* Staff acknowledgment of assignments is handled through the screen display and voice callout only.

---

## 20. User Roles & Permissions

|Role|Access Level|
|----|------------|
|**Superadmin** (IT Manager)|Full unrestricted access, including management of admins|
|**Admin** (IT)|All system access + manage users below their tier + system audit logs (non-repudiation). Cannot manage other admins.|
|**Manager**|All receptionist and staff functions + user management + reports page (projected revenue, analytics) + comprehensive transaction view|
|**Receptionist**|Schedule sessions only: assign therapist, massage type, and room to client; record transactions; produce treatment slips. Other reports (cutoff etc.) are auto-generated.|
|**Staff**|View-only interface — see assigned tasks and receive callouts. No account required. Very limited system permissions. For attendance tracking purposes.|

---

## 21. POS & Payment Rules

* The system **shall** include a **Cashier module** for the general version of SumiCare. The previously defined POS limitation **shall apply only** to the New Lasema Spa version of SumiCare.
* The **Cashier module shall be implemented as a standalone page**, separate from the **Orders/Services page**.
* The system **shall provide access to the Cashier, Orders, and Product views** under the **Receptionist role**.
* The system **shall support transaction processing**, including the selection and addition of products and/or services to a client transaction.
* Each completed transaction **shall automatically create and link a corresponding client service record**, where applicable.
* The system **shall maintain a transaction ledger** to record all transactions for audit, monitoring, and reporting purposes.
* The system **shall allow users to view and track payment and transaction history**.
* The system **shall support configurable status formats** appropriate to the service context (e.g., massage-based services), which may differ from medical or laboratory-based status formats.
* The system **shall allow authorized users to edit the client registration form**.
* The system **shall support the capture, storage, or referencing of payment-related data** necessary for operational tracking and reporting, in accordance with system configuration and client workflow requirements.

---

## 22. Holiday & Calendar Rules

* Holidays are significant for Lasema — **days leading up to holidays** (especially long weekends) drive high client volume.
* The system must include a **holiday/calendar function** to support **predictive analytics** and volume forecasting based on historical holiday patterns.

---

## 23. Client Feedback Rules

* A **client feedback system** (5-star rating) must be integrated into the public-facing frontend.
* **Customer reviews** are to be integrated into the SumiCare public booking website.

---

## 24. Voucher Rules

* The system must account for **vouchers** — exact rules for voucher types, redemption, and discounting are to be defined further.

---

## 25. System Availability

* SumiCare must operate **continuously, 24/7**, including holidays.
* The system must be accessible **remotely** (outside company premises) for management report access.

---

## 26. Language & Terminology Policy

* The term **"automate"** is not to be used in documentation or interfaces. Use **"computerize"** instead.
* The project title: **SumiCare: A Web-Based Spa Operations Management Platform for New Lasema Spa Jjimjilbang**.

---

*Note: All files for the images associated with the frontend of the system are kept within the attached Magic Patterns project.*
