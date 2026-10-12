# GVCW Pakistan (Greece) booking flow: recon notes

Captured 2026-10-11 from a real logged-in session in the built-in browser (Islamabad VAC, Long-Term Type D Seasonal/Dependent Employment). Read only: OTP and Book were not clicked. No personal data stored here.

## Protection
- Site sits behind Imperva Incapsula (`/_Incapsula_Resource` challenge, "Error 15" block page). Non-Pakistani IPs blocked. Needs a real browser with JS; raw HTTP clients will be blocked.
- Google reCAPTCHA v2 (image challenge) at **login** AND a second one at **final booking** (shown after OTP is requested, inside the appointment form as `g-recaptcha-response`).

## Endpoints
| Step | Call | Notes |
|---|---|---|
| Login | `POST /api/v1/auth/login` | username/email + password + reCAPTCHA token |
| Booking type flags | `GET /api/v1/country/bookappointmentype/{email}` | returns `bookappointmentype=true`, `otp=true` |
| Slot search | `PUT /api/v1/periodslot/slots` | one date per request, JSON body below |
| Request OTP | `POST /api/v1/onetimepassword/sendOtpBookAppointment/{phone}/{prefixId}` | body `{datefrom, selectedtime}`; only works after a slot is selected; resend button locked ~2 min |
| Book | `POST /appointments` (form `#appointment`, button `#btn-new-appointment`) | needs OTP + reCAPTCHA + terms checkbox `submitinfo` |

## Slot search body (built by app.js)
```json
{"datefrom":"DD/MM/YYYY","type":26,"bookingfor":0,"members":1,"method":1,
 "travelpurposes":-1,"howmanyapplicantsareunder12":0,"appointmentId":"undefined","id":0,
 "vac":{"id":137}}
```
- type 26 = Long-Term Type D (Seasonal/Dependent Employment); 0 = Schengen C; 2 = National D; 6 = Prime Time
- bookingfor 0 = Individual, 1 = Group (Family/Traveler), members 2 to 5
- vac 137 = Islamabad; phone prefix id 197 = Pakistan +92

## Slot search response
`returnobject.slots[]`, 8 slots per day (09:30 to 11:15, 15 min each). Each has `id`, `periodid`, `starttime`, `numofavailableslots`, `isavailable`, `isselectable`. Free slot = `isavailable && isselectable`.

**Verified live 2026-10-11:** a date that is not released yet (01/11/2026) answers `code: SUCCESS` with an **empty** `slots` list. A released date answers 8 slots (all unavailable when full). The on-screen "You cannot book an appointment on this date" is UI text, not the API reply. Release detection = slot list goes from empty to 8 entries.

## Session
- **Verified 2026-10-11:** login survived 30+ minutes idle, and survived closing and reopening the browser pane (cookies persist).

## Booking form fields
type, travelpurposes, bookingfor, members, email, phonenumberprefix[id], phonenumber, applicants[][surname, firstname, dateofbirth, passportnumber, traveldocumentvaliduntil, gender[id], nationality[id], periodslotid], datefrom, selectedtime, appointmentmethod, onetimepassword, g-recaptcha-response, submitinfo. Individual booking uses the `gp_*` element IDs (gp_surname, gp_firstname, gp_dateofbirth, gp_passportnumber, gp_traveldocumentvaliduntil, gp_gender, gp_nationality); email `#ind_email`, phone `#ind_phonenumber`.

## Known constraints (from user)
- 4 applicants, family (cousins), each with own account and own phone for SMS OTP.
- Slots for next month appear to open around the 30th of each month; exact time unknown.
- Selecting a time looks client-side only; the slot appears to be reserved only on the final POST (not yet confirmed).

## Still unknown
- Real OTP SMS text / sender ID.
- Real reply to the final Book POST (code values and result page).
