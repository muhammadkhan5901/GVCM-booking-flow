# GVCW appointment helper (Islamabad VAC, Greece)

Books one appointment per family member on pk-gr-services.gvcworld.eu. You stay in the loop for the two reCAPTCHAs (login and final booking). The bot handles everything else: watching for the release, searching dates, picking distinct slots, requesting the OTP, typing the OTP it receives from each phone, and clicking Book.

## How it works

1. `launch` opens one normal Chrome window per person, each with its own profile, so each has its own session. You log in yourself.
2. `run` attaches to those windows. Only **one** window scans dates (one request every 2 s), so the 4 accounts don't all hammer the site.
3. While the month is still closed, the bot checks 2 dates every 3 minutes and logs the exact minute it opens.
4. When free slots appear, each person's window takes a **different** slot, requests the OTP and waits for the SMS (relayed through ntfy).
5. It then beeps, pushes to your phone and brings that window to the front so you can solve the CAPTCHA. It clicks Book the moment the CAPTCHA is solved. CAPTCHAs are queued one at a time.
6. Safety: if the server's answer after Book is unclear (timeout, odd reply), that person is marked **UNCERTAIN** and is never retried automatically. Check that window yourself. This rule exists so that nobody gets booked twice.
7. If the bot crashes or you press Ctrl+C, the Chrome windows stay open and logged in. Start `run` again to continue.

## One-time setup (Windows)

1. Install **Python 3.12** from python.org and tick "Add python.exe to PATH". Google Chrome must also be installed.
2. Unzip this folder, for example to `C:\gvcw-bot`. Open PowerShell in that folder and run:
   ```
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   ```
   (No `playwright install` is needed, because the bot uses your installed Chrome.)
3. `copy config.example.yaml config.yaml`, then fill in the 4 persons exactly as on their passports.
4. Create two secret topic names and paste them into `relay.otp_topic` and `relay.notify_topic`:
   ```
   python -c "import secrets;print(secrets.token_urlsafe(24))"
   ```
   Keep them private. Anyone who knows a topic name can read it.

### Each cousin's Android phone (OTP relay)

1. Install **"SMS to URL Forwarder"** (open source, by bogkonstantin) from F-Droid or from its GitHub Releases page. It is not on Google Play.
2. Add a forwarding rule:
   * Sender: `*` for the first test. Later, set it to the GVCW sender name once we have seen a real OTP SMS.
   * URL: `https://ntfy.sh/`
   * JSON payload: `{"topic":"<your otp_topic>","message":"P1 %text%"}`. Use that person's key (P1, P2, P3, P4).
   * Do **not** add `%from%`, because a numeric sender ID could be mistaken for the code.
3. Allow SMS permission, and turn off battery optimisation for the app.
4. Test it from the PC with `python -m gvcw relay-test P1`. Then send any SMS containing a number such as `test 4321` to P1's phone. The PC should print the code.

### Your phone (alerts)

Install the **ntfy** app, subscribe to your `notify_topic`, then run `python -m gvcw notify-test`.

## Every time

```
python -m gvcw launch      # 4 Chrome windows open; log in to each (save the password in that window's Chrome)
python -m gvcw check       # every line should show form=True and a probe result
python -m gvcw fill        # fills each form; every field should say ok
python -m gvcw scan        # read-only: free slots per date of target.month
python -m gvcw run --dry-run   # full flow but stops right after selecting a slot (no OTP, no booking)
python -m gvcw run             # LIVE
```

Logs are written to `logs\bot-YYYYMMDD.log`, and screenshots of every booking attempt go to `logs\screens\`.

## Release-day plan (target month opens around the 30th)

* From the 28th: `run` with `target.month` set to the next month. It checks quietly every 3 minutes and alerts you the moment the month opens.
* Before the expected release: keep the PC on the UPS and keep a mobile hotspot ready. Re-check `check`, and confirm all 4 phones have the forwarder running with mobile data on.
* After release: you only need to watch for the beeps and solve each CAPTCHA. Every person's result is printed at the end and pushed to your phone.
* For the rest of the month, leave `run` going. It sweeps every 5 minutes for cancellations.

## Verified on the real site (11 Oct 2026)

* A date that is not released yet answers SUCCESS with an **empty** slot list. A released date answers its 8 slots, all marked unavailable when the day is full. The bot uses exactly this rule to detect the release.
* The login survived 30+ minutes idle, and even closing the browser pane. With `keepalive_every_s: 600` you can safely log in about an hour before the expected release.

## Still to confirm on the real site

* The real OTP SMS text and sender. Set `otp_regex` and the forwarder's sender filter after the first real OTP.
* The real final Book reply. Until then, anything unexpected after Book is treated as UNCERTAIN, so it can never cause a double booking.

## Tests (developer)

`pytest tests/test_unit.py` runs the unit tests. `tests/e2e_mock.py` runs the whole flow against a local mock of the site with real Chromium (Linux, headless). It covers the release moment, a competitor stealing a slot, a late server reply, and dry run.
