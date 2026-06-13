# Voicebot — Enterprise Whistleblower Reporting System

A production-grade telephony voicebot that enables employees to report workplace concerns anonymously over a phone call. Built with Google Dialogflow ES and Node.js, it handles the full reporting lifecycle: capturing the incident description, optionally collecting contact details, generating a unique case tracking code, and allowing callers to check back on their case status — all through natural voice conversation.

---

## What it does

Employees dial a dedicated hotline number. The voicebot guides them through:

1. **Privacy policy disclosure** — the bot reads out the data handling policy before any information is collected
2. **Incident recording** — the caller describes the concern in their own words; the description is saved to the database
3. **Optional registration** — callers can choose to leave contact details (name, email) or remain fully anonymous
4. **Keycode generation** — a unique 16-digit code is generated and read back digit-by-digit so the caller can note it down
5. **Status check** — callers can phone back at any time, provide their keycode, and hear the current status of their case

---

## Voice pipeline

```
Caller dials in
      │
      ▼
Google Contact Center AI (CCAI) / Dialogflow Telephony
      │  Speech-to-Text
      ▼
Dialogflow ES — NLP intent classification
      │  Matched intent + extracted parameters
      ▼
Webhook (Node.js / Express) — fulfillment handler
      │  Business logic (generate keycode, persist report, check status)
      ▼
MongoDB — case storage (description, registration, status, keycode)
      │
      ▼
Webhook response (SSML)
      │  Text-to-Speech
      ▼
Caller hears synthesised voice response
```

---

## Tech stack

| Layer | Technology |
|---|---|
| NLP / Voice | Google Dialogflow ES |
| Telephony | Google CCAI / Dialogflow Phone Gateway |
| Speech synthesis | SSML (Speech Synthesis Markup Language) |
| Webhook server | Node.js + Express |
| Database | MongoDB + Mongoose |
| Runtime | Node.js 16+ |
| Dev server | nodemon |
| Tunnelling (local dev) | ngrok |

---

## Key technical achievements

- **Telephony-aware SSML responses** — the webhook detects the `GOOGLE_TELEPHONY` source and returns platform-specific `telephonySynthesizeSpeech` responses; falls back to plain text for non-phone channels
- **SSML verbatim read-back** — keycodes are wrapped in `<say-as interpret-as="verbatim">` so each digit is spoken individually, preventing misheard numbers
- **Anonymous-first data model** — only `keycode` and `description` are required; name, email, and phone are optional, so callers can report without identifying themselves
- **Case lifecycle tracking** — each report has a `status` field (`NEW` → `PENDING` → `HANDLED`) that the caller can query at any time using their keycode
- **Multi-turn conversation** — Dialogflow output contexts carry state between turns (e.g. the `continue-status` context keeps the keycode in scope so the caller can replay or return to the menu)
- **16-digit keycode generation** — unique per registration, read back via SSML verbatim mode to prevent digit grouping by the TTS engine

---

## Conversation flows

### Flow 1 — New report (anonymous)

```
Bot:  Welcome. Say "Privacy policy" to hear our policy,
      "New registration" to file a report, or
      "Check my status" to track an existing case.

User: New registration

Bot:  Please describe the incident you wish to report.

User: [describes incident]

Bot:  Thank you for sharing that with us.
      Would you like to provide your personal details? (Yes/No)

User: No

Bot:  Understood. Your report has been recorded.
      Please prepare something to write down your keycode.
      Say "Ready" when you are ready to hear the number.

User: Ready

Bot:  Your key code is: 3 7 4 9 1 8 2 6 5 0 1 1 3 4 7 2
      Say "Repeat" to hear it again or "Finish" to return to the menu.
```

### Flow 2 — New report (with registration)

```
[Same as Flow 1 through incident description]

User: Yes

Bot:  Please provide your first name, last name, and email address.

User: [provides details]

Bot:  Thank you. Your registration has been recorded.
      Please prepare something to write down your keycode.
      Say "Ready" when you are ready.

[Keycode read-back as above]
```

### Flow 3 — Check case status

```
User: Check my status

Bot:  Please provide your keycode.

User: [reads keycode]

Bot:  The status of your case is: In Progress.
      Say "Replay" to hear again or "Finish" to return to the menu.
```

### Flow 4 — Privacy policy

```
User: Privacy policy

Bot:  Our privacy policy: [policy points].
      To check the status of your case, say "Check my Status".
      To continue to registration, say "New registration".
```

---

## Dialogflow intents & webhook actions

| Dialogflow action | Handler | Purpose |
|---|---|---|
| `handle-policy` | `handlePolicy` | Read privacy policy to caller |
| `handle-recording` | `handleRecording` | Save incident description to DB |
| `handle-registration` | `handleRegistration` | Save contact details to DB |
| `handle-keycode` | `handleKeycode` | Generate keycode, save to DB, read back via SSML |
| `handle-case` | `handleCase` | Look up case status by keycode |

---

## Data model

```js
Person {
  keycode:     String,   // required — unique 16-digit case identifier
  description: String,   // required — the reported incident
  firstname:   String,   // optional — caller may remain anonymous
  lastname:    String,   // optional
  email:       String,   // optional
  status:      enum('NEW', 'PENDING', 'HANDLED'),  // default: PENDING
}
```

---

## Local setup

### Prerequisites

- Node.js 16+
- MongoDB instance (local or Atlas)
- Google Cloud project with a Dialogflow ES agent
- ngrok (for exposing the local webhook during development)

### 1. Clone and install

```bash
git clone <repo-url>
cd FinallyBirabaye
npm install
```

### 2. Environment variables

```bash
cp .env.example .env
# edit .env with your MongoDB connection string
```

### 3. Start the webhook server

```bash
npm start          # starts nodemon on port 8080
npm run ngrok      # in a separate terminal — exposes localhost:8080
```

Copy the ngrok HTTPS URL (e.g. `https://abc123.ngrok.io`).

### 4. Configure Dialogflow

1. Open your Dialogflow ES console
2. Go to **Fulfillment → Webhook**
3. Set the URL to `https://<your-ngrok-url>/dialogflow`
4. Enable webhook fulfillment for each intent
5. Enable the **Dialogflow Phone Gateway** integration and assign a phone number

### 5. Test

- Use the Dialogflow simulator to test individual intents
- Call the assigned phone number to test the full telephony flow end-to-end

---

## Security

- **Anonymous by design** — personal details are never required; callers can file reports without providing any identifying information
- **Keycode-only access** — case status can only be retrieved by providing the keycode; there is no voicebot query path by name or email
- **Environment variables** — database credentials are loaded from `.env` via `dotenv`, never hardcoded
- **Input isolation** — all parameters are extracted by Dialogflow's NLP engine before reaching the webhook; the webhook reads structured `queryResult.parameters` rather than raw user speech strings
- **CORS** — applied to the admin REST API (`/persons`) for controlled dashboard access

---

## REST API (admin / compliance dashboard)

The `/persons` endpoint provides a CRUD interface for case management:

| Method | Path | Description |
|---|---|---|
| `GET` | `/persons` | List all cases |
| `POST` | `/persons` | Create a case record |
| `GET` | `/persons/:id` | Retrieve a specific case |
| `PATCH` | `/persons/:id` | Update case description |
| `DELETE` | `/persons/:id` | Remove a case |

---

## Project structure

```
FinallyBirabaye/
├── index.js          # Express server + all Dialogflow webhook handlers
├── models/
│   └── Person.js     # Mongoose schema (keycode, description, status, contact)
├── routes/
│   └── persons.js    # REST CRUD for admin case management
└── package.json

MongoIntegration/
└── index.js          # Earlier prototype — MongoDB integration experiments
```

---

## Author

Benito Ishimwe
