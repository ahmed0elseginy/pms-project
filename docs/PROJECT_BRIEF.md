# Student Housing Platform — Project Brief

## Overview

A platform that connects students looking for housing near universities in Egypt with verified property owners and brokers. The initial target is **Cairo University** and surrounding areas.

---

## User Roles

| Role | Description |
|------|-------------|
| **Admin** | The platform team — manages verification, disputes, and operations |
| **Property Owner / Broker** | Lists and manages housing units |
| **Student** | Searches for and books housing |

---

## Tech Stack

- **Frontend:** React JS (in progress)
- **Design:** Figma — [View Design](https://www.figma.com/design/wlhx7IorgOYTRPhZ291FPr/CO-.Gr?node-id=2313-1419)
- **Backend:** TBD (build backend now; server will be provisioned once the project launches)
- **Media:** Supports image and video uploads for listings

---

## MVP Scope

Build a solid MVP first, then expand features based on market feedback. All features below marked **[POST-MVP]** are planned for the first update after launch.

---

## Core Features

### 1. Identity Verification & Document Upload

**Goal:** Ensure every user on the platform is real and traceable to reduce fraud.

**Property Owners / Brokers:**
- Upload national ID photo
- Proof of ownership or management rights (title deed, notarized lease, or utility bill)
- Optional: recent police clearance certificate
- Phone number required during registration for confirmation

**Students:**
- Upload national ID photo
- University ID card / official enrollment proof *(optional — first-year students may not have one yet)*
- Specify gender (male / female)
- Specify faculty/college
- Age must fall within a defined range (enforced via frontend validation at registration)

> **Note:** University ID is optional because first-year students — the primary target audience — often don't have one yet in their first semester. National ID + age range is sufficient for MVP verification.

---

### 2. Listing Verification — "Verified Housing" Badge

**Goal:** Eliminate the gap between listing photos and reality.

- Property owner or broker uploads a clear, detailed video walkthrough of the unit
- OR a platform representative visits and inspects the property in person
- Verified listings receive a visible **"Verified" badge** on the platform
- Unverified listings are still shown but clearly marked as unverified

---

### 3. Property Listings & Smart Recommendations

**All Listings Page:**
- Browse all approved and verified listings on the platform

**"For You" Page (Smart Feed):**
Personalized recommendations based on student profile:
- Female-only housing for female students
- Male-only housing for male students
- Faculty-specific housing (e.g., near Medical, Engineering, Commerce faculties)
- Suggestions filtered by university, city/area, or budget

---

### 4. Listing Detail Page — Map & Nearby Services

Each listing includes:
- Full property details (rooms, beds, price, amenities, rules)
- Photos and videos
- **Interactive map** showing:
  - Exact property location
  - Nearby public transport
  - Restaurants and cafés
  - Pharmacies and supermarkets
  - Other key services around the property

> Especially important for students relocating from other governorates who don't know the area.

---

### 5. Booking & Secure Online Payment

**Booking flow (similar to e-commerce checkout):**

1. Student pays through the platform — **never directly to the owner**
2. Platform holds the payment
3. Student moves in and confirms everything matches the listing
4. After a **48-hour hold**, funds are released to the property owner
5. If there's a problem or fraud, the student gets a full refund immediately

**Supported Egyptian payment methods:**
- Vodafone Cash
- Fawry
- InstaPay
- Visa / Meeza

**Platform revenue model:**
- Commission on every successful booking/rental
- Remaining amount transferred to the property owner after the hold period
- Can be automated later via AI and payment automation systems

---

### 6. Contact Info Revealed After Booking

- Before booking: all communication stays inside the platform chat only
- After confirmed booking: the student receives the owner's/broker's phone number
- Optionally: an automatic WhatsApp message is sent to finalize move-in details

---

### 7. Property Owner / Broker Dashboard

Owners can list a unit and specify:
- Target gender (male / female)
- Target faculty (optional)
- Number of rooms and beds
- Monthly price
- Amenities and utilities
- Photos and videos
- House rules and conditions

Listing goes live after admin review and verification.

---

### 8. Security Deposit

- Student pays a small security deposit at the time of booking, held by the platform
- If the student leaves the unit in good condition → full deposit refunded
- If damage is found → owner is compensated based on documented evidence (photos/report)

---

### 9. In-Platform Chat — Privacy First

- All pre-booking communication happens through the platform's built-in chat
- **No phone numbers or personal data are visible** to either party before a confirmed booking
- Especially important for protecting female students' privacy

---

### 10. Mutual Review System

**After the stay:**
- Student reviews the housing and owner
- Owner reviews the student (reliability, respect for property)

**Instant ban policy:**
- Any owner or user with repeated complaints or safety-related bad reviews is **permanently banned immediately**

---

### 11. Emergency Button & Fast Support (SOS)

Available in the student dashboard:
- Prominent SOS / emergency complaint button
- Platform support team intervenes immediately on urgent issues
- Emergency contact numbers (ambulance, police)
- Fast support channels
- Real-time case tracking for urgent situations

---

## Additional Features (Proposed)

### "One-Night Stay" Feature *(Discussed for POST-MVP)*
- Extend the platform beyond students to anyone needing temporary accommodation (teachers, doctors, travelers)
- Opens a new user segment beyond the core student audience
- **Decision:** Launch MVP first, add this in the first update based on market demand

### "Exam Month Housing" Feature *(Discussed for POST-MVP)*
- Dedicated section for short-term (one month) rentals during exam season
- Targeted notifications for students who need temporary study housing
- Solves a real and recurring pain point for university students
- **Decision:** Launch MVP first, assess demand post-launch

---

## Open Questions / Decisions Needed

| Question | Status |
|----------|--------|
| Exact age range allowed for student registration | ❓ To be decided |
| Which faculties to support at launch | ❓ To be decided |
| Manual vs. automated admin verification workflow | ❓ To be decided |
| Commission percentage per booking | ❓ To be decided |
| Whether to show unverified listings at all in MVP | ❓ To be decided |
| Dark mode support | ✅ Planned (mentioned as in progress) |

---

## Business Model

- **Primary revenue:** Commission on every successful booking
- **Future options:** Premium listings, subscriptions for owners, advertising
- **Team structure:** Equity-based partnership — revenue split among team members, no upfront salaries
- **Server:** Will be provisioned once the project launches; backend should be built and ready before then

---

## Target Market

- **Primary city:** Cairo
- **Primary audience:** Students enrolling in or attending Cairo University and nearby institutions
- **Key focus:** First-year students (most likely to need housing, least likely to have a university ID yet)
- **Secondary audience (post-MVP):** Anyone needing short-term accommodation in the same areas
