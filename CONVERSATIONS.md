# Conversation Flows

Full example conversations for the enterprise whistleblower reporting voicebot.
All flows are telephony-based — the caller hears synthesised speech (SSML) and responds verbally.

---

## Flow 1 — Anonymous report (no personal details)

The most privacy-preserving path. Only the incident description is stored.

```
BOT:   Welcome to the reporting hotline. You can say:
       "Privacy policy" to hear how we handle your data,
       "New registration" to file a new report, or
       "Check my status" to track an existing case.

USER:  New registration

BOT:   Please describe in your own words the incident or concern
       you would like to report.

USER:  I witnessed my manager approving invoices from a vendor
       that is owned by his brother-in-law, without disclosing
       the relationship.

BOT:   Thank you for sharing that with us.
       Would you like to provide your personal details? Yes or No.

USER:  No

BOT:   Understood. Your report has been recorded anonymously.
       Please prepare something to write down your key code —
       you will need it to check the status of your case later.
       Say "Ready" when you are ready to hear the number.

USER:  Ready

BOT:   Your key code is:
       3 — 7 — 4 — 9 — 1 — 8 — 2 — 6 — 5 — 0 — 1 — 1 — 3 — 4 — 7 — 2
       Say "Repeat" if you would like me to read it again,
       or say "Finish" to return to the main menu.

USER:  Repeat

BOT:   Your key code is:
       3 — 7 — 4 — 9 — 1 — 8 — 2 — 6 — 5 — 0 — 1 — 1 — 3 — 4 — 7 2
       Say "Repeat" to hear it again or "Finish" to return to the menu.

USER:  Finish

BOT:   Thank you. Returning to the main menu.
       You can say "Privacy policy", "New registration", or "Check my status".
```

**Data saved to DB:**
```json
{
  "keycode": "3749182650113472",
  "description": "I witnessed my manager approving invoices...",
  "status": "PENDING"
}
```

---

## Flow 2 — Report with voluntary registration

The caller opts to leave contact details so compliance officers can follow up.

```
BOT:   Welcome to the reporting hotline. You can say:
       "Privacy policy", "New registration", or "Check my status".

USER:  New registration

BOT:   Please describe the incident or concern you would like to report.

USER:  There are safety violations being ignored on the factory floor.
       Workers are not being provided with required protective equipment.

BOT:   Thank you for sharing that with us.
       Would you like to provide your personal details? Yes or No.

USER:  Yes

BOT:   Please provide your first name, last name, and email address.

USER:  My name is Jane Doe. My email is jane.doe@company.com.

BOT:   Thank you, Jane. Your registration has been recorded.
       Please prepare something to write down your key code.
       Say "Ready" when you are ready to hear the number.

USER:  Ready

BOT:   Your key code is:
       8 — 1 — 2 — 3 — 4 — 5 — 6 — 7 — 8 — 9 — 0 — 1 — 2 — 3 — 4 — 5
       Say "Repeat" to hear it again or "Finish" to return to the menu.

USER:  Finish
```

**Data saved to DB:**
```json
{
  "keycode": "8123456789012345",
  "description": "There are safety violations being ignored on the factory floor...",
  "firstname": "Jane",
  "lastname": "Doe",
  "email": "jane.doe@company.com",
  "status": "PENDING"
}
```

---

## Flow 3 — Check case status

A caller who previously filed a report phones back to check progress.

```
BOT:   Welcome to the reporting hotline. You can say:
       "Privacy policy", "New registration", or "Check my status".

USER:  Check my status

BOT:   Please provide your key code.

USER:  3 7 4 9 1 8 2 6 5 0 1 1 3 4 7 2

BOT:   The status of your case is: In Progress.
       Say "Replay" to hear this again,
       or say "Finish" to return to the main menu.

USER:  Replay

BOT:   The status of your case is: In Progress.
       Say "Replay" to hear this again,
       or say "Finish" to return to the main menu.

USER:  Finish

BOT:   Returning to the main menu.
```

**Status values the caller may hear:**
| DB value | Spoken to caller |
|---|---|
| `NEW` | Case received, pending review |
| `PENDING` | In Progress |
| `HANDLED` | Your case has been resolved |

---

## Flow 4 — Privacy policy disclosure

A cautious caller asks to hear the data policy before deciding whether to report.

```
BOT:   Welcome to the reporting hotline. You can say:
       "Privacy policy", "New registration", or "Check my status".

USER:  Privacy policy

BOT:   Our privacy policy:
       1. Your report is treated with strict confidentiality.
       2. Personal details are optional — you may report anonymously.
       3. Your data is stored securely and accessed only by authorised compliance personnel.
       4. You may request deletion of your personal data at any time.
       To check the status of your case, say "Check my Status".
       To continue to registration, say "New registration".

USER:  New registration

[continues into Flow 1 or Flow 2]
```

---

## Error / fallback handling

```
USER:  [says something not recognised]

BOT:   I'm sorry, I didn't catch that.
       You can say "Privacy policy", "New registration", or "Check my status".
```

```
USER:  [provides invalid or unknown keycode]

BOT:   I was unable to find a case with that key code.
       Please check the number and try again,
       or say "Finish" to return to the main menu.
```
