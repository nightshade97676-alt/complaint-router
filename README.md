# Complaint Router

**An AI-powered civic complaint dashboard for local government in Tamil Nadu.**

A citizen describes a neighbourhood problem in plain language (English, Tamil, Malayalam and more). The app works out which office should handle it, rates how urgent it is, drafts a formal complaint letter, and saves the complaint so officials can track it from an admin dashboard.

**Live app:** _add your Render link here_
**Repository:** _add your GitHub link here_

---

## What it does

**For citizens**
- Describe the problem in your own words. Only the description is required.
- Optional details: area, landmark, pincode, photo, contact (email or phone), and "Use my location" (adds a map link).
- Gemini reads the complaint and returns a department, a severity (High / Medium / Low), an urgency score from 1 to 10, a summary and a suggested action.
- A formal letter addressed to the right office is drafted automatically. It can be edited, copied or downloaded.
- A "Where to follow up" box shows official channels for that department.
- The page can be switched to Tamil, Hindi, Malayalam or Telugu.

**For officials (admin page)**
- Totals, severity counts, a breakdown by department and by status.
- Filters: text search, department, severity, status and date range.
- Change a complaint's status (New, In Progress, Resolved) and open a detail view.
- A skyline banner whose buildings show how many complaints each department has.

**Design**
- Works on desktop and phone.
- The skyline at the top doubles as the loading screen: an envelope hops between buildings while the AI works.

---

## Departments (Tamil Nadu)

Complaints are routed the way civic issues are handled across Tamil Nadu.

| Department | What it handles |
|---|---|
| Local Body - Engineering | Potholes, road obstruction or fallen trees, streetlights, water leakage, sewage overflow |
| Local Body - Sanitary & Public Health | Garbage not collected, drain blockage, stray animals |
| Local Body - Commissioner's Office | Park maintenance and general complaints |
| TNPDCL (Electricity) | Power outages and electricity problems |
| Tamil Nadu Police (and Traffic Police) | Illegal parking, traffic issues, noise |

Notes:
- "Local body" means the Municipal Corporation, Municipality or Town Panchayat. Urban bodies come under the Municipal Administration and Water Supply Department and rural ones under Rural Development and Panchayat Raj.
- Officer titles differ by type of body: a Corporation has a Commissioner, City Engineer and City Health Officer; a Municipality has a Commissioner, Municipal Engineer and Sanitary Officer; a Town Panchayat is run by an Executive Officer. For a village panchayat, contact the panchayat office.
- State highways belong to the Highways Department (report them on the Namma Saalai app) and national highways to NHAI. Neither is a department in this app.
- TNPDCL is the Tamil Nadu Power Distribution Corporation, formerly TANGEDCO.

### Follow-up channels shown to citizens

| Channel | Used for |
|---|---|
| State CM Helpline `1100` (Mudhalvarin Mugavari) | Local body and police complaints |
| `tnurbantree.tn.gov.in` | Finding your town's office page |
| TNPDCL Minnagam `94987 94987` or `1912` | Electricity |
| `112` | Emergencies |

These come from official Tamil Nadu pages and news reports of the launches. Contact details change, so check them before relying on them. The app drafts a letter but does not send it.

---

## Tech stack

| Part | Technology |
|---|---|
| Backend | Python, FastAPI, Uvicorn |
| Database | SQLite (single file) |
| Frontend | HTML, CSS and vanilla JavaScript (no framework, no build step) |
| AI | Google Gemini through the `google-genai` library, structured JSON output |
| Hosting | Render (free tier) |

The skyline is drawn with JavaScript and SVG inside the pages, so there are no image files to manage.

---

## Project structure

```
complaint-router/
├── main.py              # FastAPI app: routes, Gemini calls, validation, database
├── templates/
│   ├── index.html       # Citizen page (form, loading screen, result)
│   └── admin.html       # Admin dashboard
├── static/
│   └── style.css        # All styling
├── requirements.txt     # Python dependencies
├── render.yaml          # Render deployment settings
├── .gitignore
└── README.md
```

The `data/` folder (database and uploaded photos) is created when the app first runs and is not committed.

---

## Run locally

You need Python 3.12 or newer and a Gemini API key from Google AI Studio.

```bash
git clone <your-repo-url>
cd complaint-router
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS / Linux
pip install -r requirements.txt
```

Create a file named `.env` in the project folder (never commit it):

```
GEMINI_API_KEY=your_key_here
ADMIN_PASSWORD=choose_a_strong_password
```

Start the app:

```bash
uvicorn main:app --reload
```

- Citizen page: http://127.0.0.1:8000
- Admin page: http://127.0.0.1:8000/admin (asks for `ADMIN_PASSWORD`)

### Try it without using AI quota

Set `MOCK_AI=1` before starting. The app then returns sample analyses instead of calling Gemini. This is for development only.

---

## Environment variables

| Name | Required | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | Yes | Google Gemini API key |
| `ADMIN_PASSWORD` | Yes | Password for the admin dashboard. If unset, the app falls back to `admin123` and logs a warning, so always set your own. |
| `GEMINI_MODEL` | No | Overrides the default Gemini model |
| `DATA_DIR` | No | Folder for the database and uploads (default `./data`) |
| `MOCK_AI` | No | Set to `1` to use sample AI output (development only) |

---

## How it works

1. The citizen submits the form to `POST /api/complaints`.
2. The server checks the input (description 10 to 2000 characters, 6-digit pincode, valid email or phone, photo JPG, PNG or WEBP up to 5 MB).
3. Gemini analyses the text. The complaint text is treated as data inside the prompt, not as instructions.
4. The AI output is validated against the allowed categories, severities and score ranges before anything is saved.
5. The category is mapped to a department, the letter is built, and the complaint is stored in SQLite.
6. If the AI fails, the user sees an error and nothing is saved.

If the main model fails, the app tries a backup model.

---

## API

| Method | Path | Access | Purpose |
|---|---|---|---|
| GET | `/` | Public | Citizen page |
| GET | `/admin` | Public page, password for data | Admin page |
| GET | `/health` | Public | Health check for Render |
| GET | `/api/meta` | Public | List of departments and statuses |
| POST | `/api/complaints` | Public | Submit and analyse a complaint |
| POST | `/api/translate` | Public | Translate the interface (limited to 12 calls per IP per 10 minutes) |
| GET | `/api/track/{id}` | Public | Look up a complaint's status by ID |
| GET | `/api/complaints` | Admin | List complaints with filters |
| PATCH | `/api/complaints/{id}/status` | Admin | Change a complaint's status |
| GET | `/api/stats` | Admin | Totals and breakdowns for the dashboard |

Admin requests send the password in an `x-admin-key` header.

---

## Deploy on Render

1. Push the project to GitHub.
2. On Render, create a **Web Service** from the repository. `render.yaml` already has the settings:
   - Build command: `pip install -r requirements.txt`
   - Start command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
   - Health check path: `/health`
3. In the Render dashboard, set `GEMINI_API_KEY` and `ADMIN_PASSWORD`.
4. Every push to GitHub redeploys the app automatically.

---

## Limitations

- **Free-tier storage:** on Render's free plan the SQLite database and photos are wiped on every redeploy, and the service sleeps when idle (the first visit can take about 30 seconds). Use a persistent disk (`DATA_DIR`) or a hosted database for permanent data.
- **Gemini quota:** the free Gemini quota is small and is shared by complaint analysis and translation. When it runs out the app shows "The AI service is busy".
- **Letters are not sent:** the app only drafts the letter. The citizen copies or downloads it.
- **Contact details can go out of date:** see the note under the departments section.

---

## Security notes

- Never commit `.env` or your API key. `.gitignore` already excludes `.env`, `.venv/` and `data/`.
- Set a strong `ADMIN_PASSWORD`.
- Uploaded photos are limited by type and size.

---

## Sources for the department information

- [Cumbum Municipality contact page](https://www.tnurbantree.tn.gov.in/cumbum/contact-us-2/)
- [Tambaram Corporation general administration page](https://www.tnurbantree.tn.gov.in/tambaram/general-administration/)
- [Town Panchayat about page](https://etownpanchayat.com/AboutUs.aspx)
- [Local government in Tamil Nadu](https://en.wikipedia.org/wiki/Local_government_in_Tamil_Nadu)
- [DT Next: TNPDCL services and complaints](https://www.dtnext.in/explainer/dt-next-guide-high-eb-bill-power-cut-heres-your-complete-guide-to-tnpdcl-services-complaints-and-bill-payment)
- [DT Next: TANGEDCO renamed TNPDCL](https://www.dtnext.in/amp/story/news/tamilnadu/union-govt-approves-tangedco-renaming-to-tnpdcl-794611)
- [DT Next: Mudhalvarin Mugavari department](https://www.dtnext.in/news/tamilnadu/new-dept-mudhalvarin-mugavari-formed)
- [DT Next: 1100 CM Helpline](https://www.dtnext.in/news/tamilnadu/tn-cm-launches-1100-helpline-for-redressal-of-people-grievances)
- [Citizen Matters: Namma Saalai app](https://citizenmatters.in/namma-saalai-app-potholes-highways-repair-omr-road-infrastructure-chennai/)
