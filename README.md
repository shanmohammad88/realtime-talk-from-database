The **Realtime AI Assistant** is a small, self-hosted demo that turns spoken questions into live AI answers backed by your **SQL database**. It combines three parts:

1. **Client (browser)** — captures your voice using the browser’s SpeechRecognition API, sends the recognized text to the backend, and plays back AI replies using browser SpeechSynthesis (so you actually hear the assistant).
2. **Server (Node.js + WebSocket)** — receives the transcript, decides whether the query needs a database lookup, runs a safe parameterized SQL query (Postgres by default), injects the DB result into the prompt, calls the **OpenAI Responses API**, and returns the AI text back to the browser.
3. **Database (SQL)** — stores factual data (example: `products` table). The server reads from this table and supplies authoritative facts to the model so it replies with accurate, up-to-date info instead of hallucinating.

Data flow (quick):
User speaks → browser STT → sends text to server via WebSocket → server optionally queries DB → server calls OpenAI with DB results included → server sends AI text back → browser TTS speaks the answer.

Key features included in the ZIP:

* `server.js` — ready-to-run Node backend (uses `pg` for Postgres).
* `public/index.html` — browser UI with mic button (uses SpeechRecognition + SpeechSynthesis).
* `init_db.sql` — creates `products` table and inserts sample rows.
* `.env.example` and README with instructions.

---

# Step-by-step: how to set up and use (Windows-focused + universal commands)

> Follow each step in order. I include commands for PowerShell and psql where relevant.

## Prerequisites

* Windows PC (instructions assume PowerShell), macOS or Linux also fine.
* Install **Node.js (LTS)** — includes `npm`. Download & run installer from nodejs.org. Make sure “Add to PATH” is checked.
* Install **PostgreSQL** (or have access to a Postgres instance). Install from postgresql.org or use PgAdmin.
* Browser: **Chrome** or **Edge** (SpeechRecognition works best here).
* An **OpenAI API key** (keep this secret and store in `.env`).

---

## 1) Download & unzip the project

* You already downloaded `realtime_ai_sql.zip`. Unzip it to a folder, e.g.:

  ```
  C:\Users\Shan\Downloads\realtime_ai_sql
  ```
* Open PowerShell and `cd` into the folder that contains `package.json` and `server.js`:

  ```powershell
  cd "C:\Users\Shan\Downloads\realtime_ai_sql\realtime_ai_sql"
  ```

---

## 2) Install Node.js & verify `npm`

If PowerShell says `npm` not recognized, install Node.js (LTS) and then:

```powershell
node -v
npm -v
```

You should get versions. If not, restart your machine after installation.

---

## 3) Install project dependencies

From the project folder:

```powershell
npm install
```

If this fails with `npm` not found, re-check Node installation.

---

## 4) Create the Postgres database and run SQL

* Create a database (example uses `mydb`):

  * Using `psql` (adjust username/host/port as needed):

    ```powershell
    psql -U postgres -c "CREATE DATABASE mydb;"
    ```
  * Or create via PgAdmin.

* Run the included SQL to create `products` table and sample rows:

  ```powershell
  psql "postgres://dbuser:dbpass@localhost:5432/mydb" -f init_db.sql
  ```

  Replace `dbuser`, `dbpass`, `localhost`, and `mydb` with your actual credentials/host if different.

* Quick check:

  ```powershell
  psql "postgres://dbuser:dbpass@localhost:5432/mydb" -c "SELECT * FROM products;"
  ```

  You should see the sample rows (`Laptop`, `Phone`, `Tablet`).

---

## 5) Create `.env` and add secrets

Copy `.env.example` to `.env` and fill values:

```powershell
copy .env.example .env
notepad .env
```

Edit `.env`:

```
OPENAI_API_KEY=sk-...
DATABASE_URL=postgres://dbuser:dbpass@localhost:5432/mydb
PORT=3000
WS_PATH=/ws
OPENAI_MODEL=gpt-4.1-mini
```

**Never share** `OPENAI_API_KEY`. Keep `.env` out of public repos.

---

## 6) Start the server

From project folder:

```powershell
npm start
# or
node server.js
```

You should see logs:

* `HTTP server listening on http://localhost:3000`
* `WebSocket server running on ws://localhost:3000/ws`

---

## 7) Open the client and test (browser)

* Open: `http://localhost:3000`
* Click **Start** → browser will begin listening. Speak a test phrase:

  * “What’s the price of Laptop?”
  * “How much is Phone?”
* The client sends the recognized text to the backend. The backend will:

  * detect the word “price” → query `products` table for the product (crude heuristic uses last word),
  * inject DB result into the prompt,
  * call OpenAI Responses API and return a human-like answer,
  * browser will speak the answer via SpeechSynthesis.

Expected sample output (audio + text bubble): “The price of Laptop is \$1200. 15-inch laptop, 16GB RAM.”

---

## 8) Sample phrases to try

* “What’s the price of Laptop?”
* “Price of Phone”
* “Tell me about Tablet”
* “Do you have a Laptop in stock?” (server fallback: model answers using DB if appropriate)

---

## 9) Troubleshooting tips

* `npm` not recognized → Install Node.js and restart shell.
* **DB connection failed** → check `DATABASE_URL` in `.env` and ensure Postgres is running and accessible.
* **OpenAI errors** → verify `OPENAI_API_KEY` in `.env` and that the model name is valid. Check logs printed in terminal by `server.js`.
* **Browser message: SpeechRecognition not supported** → try Chrome or Edge.
* If server returns `"OpenAI API error"` check the full server log for status & body response from OpenAI.
* To see server console logs, run `node server.js` directly (not background) so you can read errors.

---

## 10) Customization & next steps

* **Better entity extraction**: replace the crude “last word” product detection with a simple NER or pattern matching, or use OpenAI function-calling to extract product names reliably.
* **Use MySQL**: swap `pg` with `mysql2` and change queries accordingly.
* **Realtime WebRTC**: replace browser STT/TTS with OpenAI Realtime (webRTC) for fully streamed audio in/out. This requires creating ephemeral tokens server-side.
* **Security/production**: add HTTPS, authentication, rate limiting, and move DB credentials to a secure secrets manager. Do not expose OpenAI API key on the client.
* **Caching**: use Redis for frequent queries to reduce DB load and latency.


