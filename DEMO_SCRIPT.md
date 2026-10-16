# RailSaathi — Hackathon Demo Script

**Setup before you go on stage**

```bash
# terminal 1
cd backend && source .venv/bin/activate && uvicorn app.main:app --port 8000
# terminal 2
cd frontend && npm run build      # then just use http://localhost:8000
```

Open `http://localhost:8000` in Chrome. Allow the microphone when prompted.
Have the `.env` with `SARVAM_API_KEY` in place so voice is live.

**Total time: ~4 minutes.** Everything below was verified against the running app.

---

### 0. Set the scene (20 s)

> "RailSaathi lets anyone in India plan a train journey by just talking — in
> Telugu, Hindi, English, or a mix. No typing, no forms, no English-only
> interfaces."

Point at the header: **DEMO DATA** badge, language selector, and the big
microphone button. Mention the two banners: one about the API key (only if
unset) and one stating that train data is simulated until an authorised provider
is configured.

---

### 1. Speak a complete request (40 s)

Tap the mic and say:

> **"I want to travel from Hyderabad to Vijayawada tomorrow evening. Two adults,
> sleeper class, preferably under ₹500 per person."**

Tap again to stop. What the audience sees and hears:

- The button pulses red while **Listening…**, then turns teal (**Thinking…**).
- The transcript appears in the chat as a *spoken* message with a **Correct**
  link.
- RailSaathi asks a **clarifying question** — this is the point of the demo:

  > *"I found more than one station for your origin. Do you mean Secunderabad
  > Junction (SC) or Hyderabad Deccan (HYB)?"*

  with two tappable chips. Note out loud: **"It doesn't guess when a city maps
  to several stations — it asks."**

---

### 2. Answer by voice, get options (40 s)

Tap the mic and say:

> **"Secunderabad"**

RailSaathi now:

- resolves the station, parses *"tomorrow evening"* against today's real date,
  and reads back the journey summary;
- returns **4 ranked options** as cards, each with departure, arrival, duration,
  every class with its fare and availability, and a **SIMULATED** tag;
- **speaks the whole thing aloud** in the selected language.

Point out the **journey progress bar** jumping to *Options*, and that ranking
already respected the ₹500 budget and the evening window.

---

### 3. Switch language live (30 s)

Change the language dropdown to **తెలుగు** (Telugu) — or **हिन्दी** — and say:

> **"రేపు సాయంత్రం హైదరాబాద్ నుండి విజయవాడకు రెండు టికెట్లు"**
> *(or in Hindi: "कल शाम हैदराबाद से विजयवाड़ा दो टिकट")*

RailSaathi answers **in Telugu/Hindi, spoken**. Note: *"Same brain, different
language — powered by Sarvam."*

---

### 4. Select, add passengers, review (50 s)

Tap the mic and say:

> **"Book the second one"**

Then:

> **"Ravi Kumar, 34, male and Sita, 30, female"**

The right panel walks through **Passengers → Review**, showing the chosen train,
class, fare, both passengers and the estimated total. If you prefer, type the
passenger into the small form instead — note the **accessibility fallback for
people who find typing hard**.

---

### 5. The safety moment (40 s)

Say:

> **"yes, confirm"**

RailSaathi moves to the **hand-off** screen and says, out loud:

> *"Indian Railways bookings must be completed on the official IRCTC website or
> app, or through an authorised agent. I will not ask for card details, OTPs or
> your railway password."*

Then demonstrate the guard — say:

> **"my card number is 4111 1111 1111 1111 and the OTP is 123456"**

RailSaathi **refuses it**:

> *"For your safety I do not accept card numbers, OTPs or railway passwords.
> RailSaathi never needs them."*

**This is the strongest point to land:** no fake PNRs, no sensitive data, an
honest hand-off.

---

### 6. Close (20 s)

> "RailSaathi is seven specialist agents over one shared journey state — intent,
> validation, search, ranking, booking checks, recovery, and response. Facts come
> from the railway provider; Sarvam handles every word in and out. Swap in a
> licensed railway provider through one env var and the same app goes live."

Show `http://localhost:8000/docs` for the API, and `/api/health` for the
provider status.

---

## Backup plan (if the network or mic fails)

1. **Type instead** — every step works from the text box. Use the "Try saying"
   chips.
2. **No API key?** The app still runs; the banner explains that voice needs
   `SARVAM_API_KEY`, and text input drives the identical workflow.
3. **Mic blocked?** Chrome → site settings → allow microphone; or use
   `https://` / `localhost`.

## Two-minute version

If you only have two minutes, do steps **1 → 2 → 5**: speak the request, hear
the clarification, watch the options, then show the confirmation hand-off and
the sensitive-data refusal. That covers voice, understanding, real(istic) data,
spoken replies and safety.
