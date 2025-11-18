# ISAC Radar - Gmail Alert Setup Guide

## Overview
Your ISAC obstacle detection system is now configured to send email alerts via Gmail whenever an obstacle is detected.

## Quick Start (3 Steps)

### Step 1: Enable 2-Step Verification on Your Google Account
1. Go to: https://myaccount.google.com/security
2. In the left menu, find "How you sign in to Google"
3. Click **2-Step Verification** and follow the prompts
4. Choose your recovery phone or email

### Step 2: Generate an App Password
1. Go to: https://myaccount.google.com/apppasswords
2. You may need to re-authenticate with your password
3. Select dropdown: **Mail** and **Windows Computer**
4. Click **Generate**
5. Google will show a 16-character password like: `xxxx xxxx xxxx xxxx`
6. **Copy this password** (you'll use it once)

### Step 3: Configure in Your Notebook

#### Option A: Python Cell (Notebook)
In the **Gmail Configuration** cell, update:
```python
YOUR_GMAIL = "your-email@gmail.com"
YOUR_APP_PASSWORD = "xxxx xxxx xxxx xxxx"  # Paste the 16-char password
```

#### Option B: Environment Variables (PowerShell)
Run in PowerShell before starting the notebook:
```powershell
$env:EMAIL_SENDER = "your-email@gmail.com"
$env:EMAIL_RECEIVER = "your-email@gmail.com"  # Can be different
$env:SMTP_USER = "your-email@gmail.com"
$env:SMTP_PASS = "xxxx xxxx xxxx xxxx"  # Paste the 16-char password
$env:SMTP_SERVER = "smtp.gmail.com"
$env:SMTP_PORT = "587"
```

## Testing Your Setup

1. **Run Cell 2** (Gmail Configuration)
   - Updates GMAIL_CONFIG with your credentials
   
2. **Run Cell 3** (Quick Setup)
   - Configures the settings
   
3. **Run Cell 4** (Test Gmail Configuration)
   - Tests connection to Gmail
   - Sends a test email to verify everything works
   - Shows detailed success/error messages

If test succeeds: ✓ Ready to run the main pipeline!

## Email Alert Details

### What You'll Receive
When an obstacle is detected, you'll get an email with:
- **Subject**: `[ALERT] Railway Obstacle Detected` or `[TEST] ISAC Radar - Gmail Alert System Ready`
- **Body**: Detection details including:
  - Object type (person, car, truck, bicycle, dog, etc.)
  - Confidence level (0.0 - 1.0)
  - Track ID
  - Detection coordinates
  - Timestamp

### Alert Triggers
- **Standard Alert**: Any object matching alert labels detected
- **Emergency Alert**: Confidence level ≥ 0.85 (immediate send)
- **Frequency**: One email per unique track (not spam-y)

## Troubleshooting

### "Authentication Error"
**Problem**: `SMTPAuthenticationError`
- ✓ Gmail address is correct
- ✓ App Password is 16 characters (not your regular password)
- ✓ 2-Step Verification is enabled
- ✓ Try copying/pasting the password again (no typos)

### "Connection Timeout"
**Problem**: Cannot connect to Gmail SMTP
- ✓ Check internet connection
- ✓ Verify firewall isn't blocking port 587
- ✓ Check if corporate network is blocking SMTP

### "Email not sent"
**Problem**: Connection works but email fails
- ✓ Receiver email address is valid
- ✓ Try different recipient email
- ✓ Check Gmail spam folder

### Can't find 2-Step Verification option
- Your Google Account might be managed by an organization
- Contact your administrator
- Or use a personal Gmail account instead

## Architecture

```
Obstacle Detection → Tracker → Confidence Check → Email Alert
     (Camera)        (Kalman)   (>= 0.5 default)   (Gmail SMTP)
```

### Email Flow:
1. Detector finds objects in camera frame
2. Tracker maintains object IDs across frames
3. If confidence > CONF_THRESH (0.45), prepare alert
4. Async thread sends email via Gmail SMTP
5. Log message shows success/failure

## Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `EMAIL_SENDER` | alerts@example.com | Gmail address sending alerts |
| `EMAIL_RECEIVER` | operator@example.com | Recipient email address |
| `SMTP_USER` | (empty) | Gmail authentication user |
| `SMTP_PASS` | (empty) | Gmail app password |
| `SMTP_SERVER` | smtp.gmail.com | Gmail SMTP server |
| `SMTP_PORT` | 587 | SMTP port (TLS) |
| `CONF_THRESH` | 0.45 | Minimum confidence for alerts |
| `EMERGENCY_CONF` | 0.85 | High confidence threshold |
| `ALERT_LABELS` | person, car, truck... | Objects that trigger alerts |

## Security Notes

- **Never share your App Password** - it grants email access
- App Password is different from your regular Google password
- If compromised, regenerate a new app password
- Passwords are stored as environment variables (not in code)
- SMTP communication uses TLS encryption (port 587)

## Next Steps

1. ✓ Enable 2-Step Verification
2. ✓ Generate App Password
3. ✓ Set credentials in notebook
4. ✓ Run test cell
5. ✓ Receive test email
6. ✓ Start main pipeline with `start_all()`

## Support

For issues:
1. Check logs in notebook output
2. Review error message in test cell
3. Verify Gmail settings at https://myaccount.google.com
4. Ensure internet connectivity

---
**ISAC Radar Alert System Ready!**
