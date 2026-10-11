# CTF Platform — Enterprise-Grade Cybersecurity Competition Engine

A modern, high-performance, and fully configurable web-based Capture the Flag (CTF) competition platform engineered for university cybersecurity clubs, student branches, and competitive hacking events.

---

## 🌟 Key Features & Innovations

### 1. Dynamic Database-Driven Challenge Management (Runtime CRUD)
- **Zero-Downtime Modifications:** Organizers can create, edit, delete, or toggle challenges at runtime without restarting the application.
- **Instant Live Synchronization:** Challenge updates, point adjustments, and flag changes are pushed instantly to connected participant browsers via Server-Sent Events (SSE).
- **Hybrid Flag Architecture:** Supports both traditional single-flag challenges ($100\%$ points upon solve) and multi-stage guided challenges (partial points awarded per solved stage flag).
- **Challenge File Attachments:** Directly upload challenge assets (PCAP captures, reverse-engineering binaries, disk images, wordlists) via the admin panel; participants can download files with one click from the challenge page.

### 2. Live Participant Proctoring & Real-Time Screen Monitoring
- **Live Screen Telemetry:** When proctoring is enabled, participant browsers stream periodic visual screen frames directly to the organizer's control center (3–4 second intervals).
- **Tab Focus & Multi-Tasking Detection:** Monitors when a team leaves the challenge tab or minimizes their browser (`🟢 Active` vs `🟡 Tab Blurred` vs `🔴 Offline`).
- **Real-Time Activity Telemetry:** Organizers can see the exact challenge each team is currently viewing, their active window status, IP address, and seconds since last heartbeat.
- **Inspect Screen Modal:** Click any participant card in the admin panel to open an expanded, auto-updating live screen feed.

### 3. Full Administrator Governance & Superpowers
- **On-the-Spot Account Provisioning:** Provision new team credentials instantly for walk-in or last-minute event attendees.
- **Instant Password Resets:** Reset any participant's password without needing their previous credentials.
- **Live Score Adjustments:** Add or deduct points directly with an organizer reason (e.g. hint penalties, rule infraction deductions, or bonus awards) visible instantly on the scoreboard.
- **Disqualification & Reactivation:** Disqualify disruptive participants immediately; disqualified accounts are locked from submissions and removed from rankings.
- **Secondary Master Token Defense:** Sensitive administrative actions require a secondary server-side master token (`CTF_ADMIN_MASTER_TOKEN`) ensuring defense-in-depth even if ordinary sessions are compromised.

### 4. Real-Time Live Scoreboard
- **Dual-Mode Live Updates:** Real-time Server-Sent Events (`/api/events/stream`) deliver sub-second ranking updates with automated fallback to 5-second HTTP polling if network interruptions occur.
- **Dynamic Leaderboard Aesthetics:** Top three teams receive gold, silver, and bronze trophies; logged-in teams see their own ranking row highlighted in glowing gold.
- **Subtabs:** Filter effortlessly between **Overall Rankings** and **Your Position**.

### 5. Concurrency & Network Hardening
- **Cloudflare Tunnel & Reverse Proxy Safe:** Correctly inspects `CF-Connecting-IP` and sanitized forwarded headers. Eliminates the critical risk where all participants sharing `127.0.0.1` are locked out due to a single team's failed login attempts.
- **Atomic Concurrency Protection:** Flag submissions and attempt increments use atomic SQLite transactions (`BEGIN IMMEDIATE`), eliminating race conditions where scripted requests bypass maximum attempt limits.
- **Append-Only Audit Triggers:** Every privileged action, login attempt, challenge edit, and backup is recorded in an SQLite trigger-protected append-only audit table.

### 6. Generic Brand Customization
- Easily rebrandable for any club, university, or organizer through simple environment variables (`CTF_EVENT_TITLE`, `CTF_ORGANIZER_NAME`, `CTF_EVENT_TAGLINE`, `CTF_LOGO_LEFT`, `CTF_LOGO_RIGHT`).

---

## 🏗️ System Architecture

```
PARTICIPANTS (Browsers)
      │
      ▼ HTTPS
[ CLOUDFLARE EDGE ]
      │
      │ Cloudflare Tunnel (cloudflared)
      ▼
[ HOST MACHINE ]
  ├── Waitress WSGI (:8080)
  │     └── Flask Core Application (app.py)
  │           ├── Auth & Session Management (auth.py)
  │           ├── Dynamic Challenge Store (challenges.py)
  │           ├── Live Proctoring & Screens Stream (/api/proctor/...)
  │           ├── Real-Time SSE Scoreboard Stream (/api/events/stream)
  │           └── Storage & Transactions (database.py)
  │                 └── SQLite (data/ctf.db - WAL Mode)
  │
  └── Challenge Files Storage (data/challenge_files/)
```

### Distributed Challenge Environment (Zone Separation)
The platform operates alongside a separately distributed virtual machine (e.g. Oracle VirtualBox `.ova` image) containing offline challenge environments, vulnerable services, and forensics datasets. This keeps the organizer's host machine isolated from participants' penetration testing traffic.

---

## 🚀 Quick Start Guide

### 1. Prerequisites
- Python 3.10+ (Tested on Python 3.12 and 3.13)
- PowerShell (Windows) or Bash (Linux / macOS)

### 2. Local Setup
Clone or navigate to the repository directory:

```powershell
# Create and activate virtual environment
python -m venv venv
.\venv\Scripts\Activate.ps1   # On Linux: source venv/bin/activate

# Install required dependencies
pip install -r requirements.txt

# Create your primary administrator account
python create_admin.py

# Launch development server
python app.py
```
Open **http://127.0.0.1:5000** in your web browser.

---

## ⚙️ Configuration & Environment Variables

Copy `.env.example` to `.env` or set environment variables in your deployment environment:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `CTF_EVENT_TITLE` | `"Cyber CTF Platform"` | Title shown on navigation bar and headers. |
| `CTF_ORGANIZER_NAME` | `"CYBERSECURITY CLUB"` | Name of host club or institution. |
| `CTF_EVENT_TAGLINE` | `"EXPLORE • SOLVE • WIN"` | Tagline displayed under the site header. |
| `CTF_LOGO_LEFT` | `"logo_left.png"` | Filename of left logo in `static/images/`. |
| `CTF_LOGO_RIGHT` | `"logo_right.png"` | Filename of right logo in `static/images/`. |
| `CTF_SECRET_KEY` | *(Random 32-byte hex)* | Secret key for Flask session cookie encryption. |
| `CTF_ADMIN_MASTER_TOKEN` | `"admin_secret_token_change_in_prod"` | Secondary master token for admin access. |
| `CTF_DATABASE_PATH` | `"data/ctf.db"` | SQLite database file path. |
| `CTF_CHALLENGE_FILES_DIR` | `"data/challenge_files"` | Directory where challenge asset uploads are stored. |
| `CTF_TRUSTED_PROXIES` | `"1"` | Enables `CF-Connecting-IP` proxy header parsing. |
| `CTF_HTTPS` | `"0"` | Set to `1` when served behind an HTTPS reverse proxy. |
| `CTF_DURATION_SECONDS` | `21600` | Competition duration (6 hours by default). |
| `CTF_ENABLE_PROCTORING` | `"1"` | Enables live participant screen and focus telemetry. |

---

## 🌐 Production Deployment Guide

### Running with Waitress (Windows Production WSGI)
Do not use `python app.py` for live events. Run with Waitress multi-threaded WSGI:

```powershell
$env:CTF_SECRET_KEY = "your-generated-long-random-secret"
$env:CTF_ADMIN_MASTER_TOKEN = "your-organizer-master-token"
$env:CTF_HTTPS = "1"
$env:CTF_TRUSTED_PROXIES = "1"

waitress-serve --listen=127.0.0.1:8080 --threads=16 app:app
```

### Running with Gunicorn (Linux Production WSGI)
```bash
export CTF_SECRET_KEY="your-generated-long-random-secret"
export CTF_ADMIN_MASTER_TOKEN="your-organizer-master-token"
export CTF_HTTPS="1"
export CTF_TRUSTED_PROXIES="1"

gunicorn -w 4 -k gthread --threads 8 -b 127.0.0.1:8080 app:app
```

### Exposing Publicly via Cloudflare Tunnel
1. Download `cloudflared` on the host machine.
2. Authenticate and create a named tunnel:
   ```bash
   cloudflared tunnel create ctf-event
   ```
3. Configure `config.yml`:
   ```yaml
   tunnel: <TUNNEL_ID>
   credentials-file: C:\Users\<User>\.cloudflared\<TUNNEL_ID>.json
   ingress:
     - hostname: ctf.youruniversity.edu
       service: http://127.0.0.1:8080
     - service: http_status:404
   ```
4. Start the tunnel:
   ```bash
   cloudflared tunnel run ctf-event
   ```

---

## 🛠️ Administrator Operations Manual

1. **Accessing Control Center:**
   - Log in using an administrator account.
   - Navigate to **⚙ Control Center** in the sidebar.
   - Enter your `CTF_ADMIN_MASTER_TOKEN` to unlock privileged controls.

2. **Managing Live Screens (Proctoring Tab):**
   - Click the **📹 Live Screens & Progress** tab.
   - Review participant focus status (`Active`, `Tab Blurred`, `Offline`).
   - Click any screen card to open the expanded full-resolution live screen view.

3. **Dynamic Challenge Authoring (Challenge Manager Tab):**
   - Click **+ Create New Challenge**.
   - Specify Title, Category, Difficulty, Section (Round), Points, Flags, Hint, and upload challenge files.
   - Changes take effect immediately without stopping the application.

4. **Participant Management (Users Tab):**
   - Add walk-in teams with **+ Create New Account**.
   - Use **Reset Pwd** to update forgotten passwords instantly.
   - Use **Score ±** to apply penalties or bonus points.
   - Use **Disqualify** to immediately lock a team out.

5. **Hot Backups & Data Exports (Event Tab):**
   - Click **🗄 Create Database Backup Now** for an online snapshot.
   - Click **⬇ Download participants_data.xlsx** for formatted spreadsheets.
   - Click **⬇ Export Full Audit Archive (ZIP)** for CSV audit records.

---

## 🧪 Automated Testing Suite

The repository includes comprehensive unit and integration tests:

```powershell
python -m unittest scratch/test_platform.py
```

Tests verify:
- Automatic SQLite schema creation and challenge seeding.
- User authentication, password rotation, and account deletion.
- Concurrency attempt limits and lockout enforcement.
- Real-time proctoring heartbeat and screen frame upload.
- Runtime challenge CRUD and flag modifications.

---

## 📜 License
Developed for cybersecurity education and competitive CTF events. Free for academic, student, and open-source use.


# CTF Platform — Installation, Execution and Deployment

## 1. Prerequisites

Ensure the following software is installed:

* Windows 10/11
* Python 3.12 or 3.13
* Windows PowerShell
* Cloudflare Tunnel (`cloudflared`) for optional Internet access

The project uses Flask, SQLite, Waitress, and other Python dependencies specified in `requirements.txt`.

### Verify Python installation

Open PowerShell and execute:

```powershell
py -0p
```

Verify that Python 3.13 or another supported version is installed.

To check the version:

```powershell
python --version
```

If `python` is unavailable, use the Python Launcher:

```powershell
py -3.13 --version
```

---

## 2. Navigate to the Project Directory

Open PowerShell and navigate to the project directory.

```powershell
cd "E:\5th sem\IEEE\CTF\my_ctf\my_ctf"
```

Replace the example path if the project is stored elsewhere.

Verify the project files:

```powershell
dir
```

The directory should contain files such as:

```text
app.py
auth.py
challenges.py
config.py
database.py
create_admin.py
requirements.txt
README.md
templates/
static/
data/
```

---

## 3. Create and Activate the Virtual Environment

A Python virtual environment isolates the project's dependencies from other Python installations.

### Create the environment

For Python 3.13:

```powershell
py -3.13 -m venv venv
```

If a compatible virtual environment already exists and is working, this step can be skipped.

**Important:** Virtual environments should not be copied between different directories or computers. If the project has been moved, recreate the environment at its current location.

### Activate the environment

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then execute:

```powershell
.\venv\Scripts\Activate.ps1
```

The terminal prompt should now begin with:

```text
(venv) PS
```

### If the existing environment is broken

Deactivate the environment if necessary:

```powershell
deactivate
```

Remove the broken environment:

```powershell
Remove-Item -Recurse -Force .\venv
```

Recreate it:

```powershell
py -3.13 -m venv venv
```

Activate it again:

```powershell
.\venv\Scripts\Activate.ps1
```

This removes the virtual environment and its installed packages, not the project source code or database.

---

## 4. Install Dependencies

Ensure the virtual environment is active.

Upgrade pip:

```powershell
python -m pip install --upgrade pip
```

Install the required dependencies:

```powershell
python -m pip install -r requirements.txt
```

Verify that the dependencies are compatible:

```powershell
python -m pip check
```

Expected output:

```text
No broken requirements found.
```

If dependency installation fails, resolve the reported error before proceeding.

---

## 5. Configure Environment Variables

The application uses environment variables for security-sensitive configuration.

Generate a random application secret:

```powershell
$env:CTF_SECRET_KEY = python -c "import secrets; print(secrets.token_hex(32))"
```

Generate a random administrator master token:

```powershell
$env:CTF_ADMIN_MASTER_TOKEN = python -c "import secrets; print(secrets.token_urlsafe(32))"
```

For local testing, disable proctoring unless it is specifically required:

```powershell
$env:CTF_ENABLE_PROCTORING = "0"
```

### Retrieve the administrator master token

The generated token is stored in the current PowerShell session.

Display it when needed:

```powershell
$env:CTF_ADMIN_MASTER_TOKEN
```

Store it securely so it can be entered into the administrator interface.

**Never publish the token, commit it to Git, or include it in screenshots or shared logs.**

The environment variables must be configured before starting the application. They are not automatically preserved when the PowerShell session closes.

If the application is restarted in a new terminal, configure the variables again using the same secret values if existing sessions and configuration depend on them.

---

## 6. Initialize the Database

The application uses SQLite for persistent competition data.

The database is located at:

```text
data/ctf.db
```

The application initializes the database during startup.

If the database does not exist, the initialization process creates the required tables and seeds the default challenges when appropriate.

### Important: Existing database

If `data/ctf.db` already exists, preserve it unless a fresh database is explicitly required.

Before resetting an existing database, create a backup:

```powershell
Copy-Item .\data\ctf.db .\data\ctf-backup.db
```

For a fresh competition database, stop the application and rename the existing database:

```powershell
Rename-Item .\data\ctf.db ctf-previous.db
```

Start the application afterward to initialize a new database.

Do not perform this operation if existing participants, submissions, scores, or other competition records must be retained.

---

## 7. Create the Administrator Account

Create the first administrator account by running:

```powershell
python create_admin.py
```

Enter the requested information:

```text
New administrator username:
New administrator password:
Confirm password:
```

Use a strong password.

The script stores the password as a hash rather than plaintext.

### Existing administrator account

If the database already contains an administrator, the script will not create another one.

In that case, use the existing administrator credentials or follow the application's administrator recovery procedure.

Keep the administrator password separate from the administrator master token.

---

## 8. Run the Application Locally

Ensure that:

1. The virtual environment is active.
2. Dependencies are installed.
3. Environment variables are configured.
4. The database is available.
5. An administrator account exists.

Start the Flask application:

```powershell
python app.py
```

The terminal should display a message indicating that the server is running.

For local development, the preferred address is:

```text
http://127.0.0.1:5000
```

Open this address in a web browser.

Keep the PowerShell window open while the application is running.

To stop the server, press:

```text
Ctrl + C
```

### Important security note

The Flask development server is intended for development and testing.

For local-only testing, configure the application to listen on `127.0.0.1` rather than `0.0.0.0`.

Do not expose the Flask development server directly to the public Internet.

---

## 9. Access the Platform

Once the application is running, the following routes can be accessed locally.

| Route             | Purpose                 |
| ----------------- | ----------------------- |
| `/`               | Homepage                |
| `/login`          | Participant login       |
| `/dashboard`      | Participant dashboard   |
| `/challenges`     | Challenge listing       |
| `/challenge/<id>` | Individual challenge    |
| `/scoreboard`     | Competition leaderboard |
| `/admin`          | Administrator interface |

Example:

```text
http://127.0.0.1:5000/admin
```

### Administrator authentication

The administrator workflow uses two separate credentials:

1. Administrator username and password.
2. Administrator master token for privileged administrative controls.

The master token must match the value configured through `CTF_ADMIN_MASTER_TOKEN`.

The backend must enforce authorization for every privileged operation.

### Participant accounts

Participant registration is restricted in the current implementation.

Create participant accounts through the authorised administrative interface before testing participant login.

---

## 10. Start the CTF Event

The application initializes a new competition in the `not_started` state.

Flag submissions are rejected until the event is started.

To begin the competition:

1. Log in using the administrator account.
2. Open the administrator control centre.
3. Verify the administrator master token.
4. Open the event controls.
5. Select the event-start action.
6. Confirm that the event status changes to running.

Then test flag submissions using a participant account.

The backend must enforce event status and timing independently of the frontend timer.

If submissions remain unavailable after the event starts, inspect the application logs and verify the event-state logic.

---

## 11. Run the Application Using Waitress

Waitress is included in the project dependencies and provides a production-oriented WSGI server suitable for Windows.

Stop the Flask development server before starting Waitress.

Ensure that the required environment variables are set in the current PowerShell session.

Start the application:

```powershell
waitress-serve --listen=127.0.0.1:8080 --threads=16 app:app
```

The application should now be available locally at:

```text
http://127.0.0.1:8080
```

Open this address in a browser and verify that the application loads.

**Use either the Flask development server or Waitress for a given application instance, not both on the same port.**

---

## 12. Expose the Application Using Cloudflare Quick Tunnel

Cloudflare Tunnel allows a locally running application to be accessed through a public HTTPS URL without conventional inbound router port forwarding.

### Install Cloudflare Tunnel

Open a separate PowerShell window:

```powershell
winget install --id Cloudflare.cloudflared
```

Verify the installation:

```powershell
cloudflared --version
```

If necessary, reopen PowerShell after installation.

### Start the local application

In the first PowerShell window, start Waitress:

```powershell
waitress-serve --listen=127.0.0.1:8080 --threads=16 app:app
```

Keep this window open.

### Start the tunnel

Open a second PowerShell window and execute:

```powershell
cloudflared tunnel --url http://127.0.0.1:8080
```

Cloudflare will attempt to establish a Quick Tunnel and display a generated public URL.

The URL will resemble:

```text
https://random-words.trycloudflare.com
```

The actual hostname will differ.

Share the generated URL with authorised participants only after the platform has passed its security and functionality tests.

Keep both the Waitress and Cloudflare Tunnel processes running throughout the event.

### Quick Tunnel limitations

* The hostname is temporary.
* A stable custom hostname is not provided.
* Availability depends on the host machine, application process, and Internet connection.
* Cloudflare Tunnel does not automatically secure application authentication or authorisation.

Quick Tunnels are appropriate for initial testing rather than a formal competition deployment.

---

## 13. Configure a Named Cloudflare Tunnel

For a competition requiring a stable hostname, use a named tunnel associated with a domain configured in Cloudflare.

### Authenticate Cloudflare Tunnel

```powershell
cloudflared tunnel login
```

Complete the browser authentication process.

### Create the tunnel

```powershell
cloudflared tunnel create ctf-platform
```

Record the generated tunnel UUID and keep the credentials file private.

### Route the hostname

Replace `ctf.example.com` with a hostname belonging to a domain configured in Cloudflare:

```powershell
cloudflared tunnel route dns ctf-platform ctf.example.com
```

### Create the tunnel configuration

Create:

```text
%USERPROFILE%\.cloudflared\config.yml
```

Example configuration:

```yaml
tunnel: YOUR-TUNNEL-UUID
credentials-file: C:\Users\YOUR-USERNAME\.cloudflared\YOUR-TUNNEL-UUID.json

ingress:
  - hostname: ctf.example.com
    service: http://127.0.0.1:8080

  - service: http_status:404
```

Replace the example values with the actual tunnel UUID, Windows username, credentials-file path, and hostname.

### Run the tunnel

Start Waitress in one terminal:

```powershell
waitress-serve --listen=127.0.0.1:8080 --threads=16 app:app
```

Start the named tunnel in another terminal:

```powershell
cloudflared tunnel run ctf-platform
```

The public application should be available at:

```text
https://ctf.example.com
```

Refer to the official Cloudflare Tunnel documentation for current configuration and troubleshooting instructions:

https://developers.cloudflare.com/tunnel/get-started/

---

## 14. Troubleshooting

### Error: PowerShell scripts are disabled

Run:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then activate the environment:

```powershell
.\venv\Scripts\Activate.ps1
```

### Error: pip cannot find Python

Recreate the virtual environment at the current project location and reinstall the dependencies.

```powershell
Remove-Item -Recurse -Force .\venv

py -3.13 -m venv venv

.\venv\Scripts\Activate.ps1

python -m pip install -r requirements.txt
```

### Error: Address already in use

Another process may already be using the selected port.

Stop the existing application instance or choose a different available port.

### Error: Administrator master token rejected

Verify that:

* The token was generated successfully.
* `CTF_ADMIN_MASTER_TOKEN` was set before the application started.
* The application is running in the expected PowerShell session.
* The token entered into the administrator interface matches the configured value.
* No configuration fallback is overriding the expected value.

Do not publish the token when requesting troubleshooting assistance.

### Error: Flag submission rejected

Check:

* Whether the competition has started.
* Whether the challenge is active.
* Whether the participant is authenticated.
* Whether the attempt limit has been reached.
* Whether the submitted flag matches the expected answer.
* Whether the backend returned an error.

Inspect the Flask or Waitress terminal for application exceptions.

### Error: Live scoreboard does not update

Check:

* Whether the SSE endpoint is reachable.
* Whether the browser console reports JavaScript errors.
* Whether the backend emits valid SSE messages.
* Whether the scoreboard endpoint returns valid JSON.
* Whether the frontend handles reconnection and polling correctly.

### Error: Public URL does not load

Verify that:

* The application is accessible locally.
* Waitress is still running.
* Cloudflare Tunnel is still running.
* The tunnel points to the correct local port.
* The host machine has Internet connectivity.

---

## 15. Pre-Event Verification Checklist

Before sharing the platform with participants, verify the following.

* [ ] Homepage loads correctly.
* [ ] Administrator login works.
* [ ] Administrator master token is verified correctly.
* [ ] Unauthorised requests to administrative endpoints are rejected.
* [ ] Participant accounts can be created.
* [ ] Participant login works.
* [ ] Challenge listings and challenge details load.
* [ ] Correct flags are accepted during an active event.
* [ ] Incorrect flags are rejected.
* [ ] Attempt limits are enforced by the backend.
* [ ] Duplicate submissions do not award duplicate points.
* [ ] Participant scores update correctly.
* [ ] Leaderboard updates work.
* [ ] Administrator statistics update correctly.
* [ ] Event timing and pause/resume controls work.
* [ ] Competition state survives application restarts.
* [ ] Database backups can be restored.
* [ ] The distributed VM has been tested.
* [ ] The challenge VM cannot access sensitive host or university resources.
* [ ] Cloudflare Tunnel works from an external network.
* [ ] Proctoring is configured appropriately and disclosed to participants if enabled.

**Deployment should proceed only after the core competition functions and access controls have been tested successfully.**
