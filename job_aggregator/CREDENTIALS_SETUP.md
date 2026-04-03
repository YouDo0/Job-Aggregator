# Credentials Setup Guide

This guide explains how to set up all the credentials needed for the Job Aggregator to work properly.

---

## 1. Gmail App Password (REQUIRED)

The system sends email digests via Gmail SMTP. You need an **App Password**, not your regular password.

### Steps:

1. **Enable 2-Factor Authentication** on your Google Account
   - Go to: https://myaccount.google.com/security
   - Find "2-Step Verification" and enable it

2. **Generate an App Password**
   - Go to: https://myaccount.google.com/apppasswords
   - Select app: "Mail"
   - Select device: "Other (Custom name)" → type "Job Aggregator"
   - Click "Generate"
   - You'll get a **16-character password** (e.g., `abcd efgh ijkl mnop`)

3. **Update .env**
   ```env
   EMAIL_USER=your_email@gmail.com
   EMAIL_PASSWORD=abcd efgh ijkl mnop
   EMAIL_FROM=your_email@gmail.com
   ```

### Why App Password?
- More secure than using your real password
- Can be revoked anytime if compromised
- Works with 2FA enabled

---

## 2. Job Portal Credentials (OPTIONAL)

Most job portals allow basic scraping **without any login**. However, some features may require authentication.

### LinkedIn
| Field | Required | Notes |
|-------|----------|-------|
| `LINKEDIN_EMAIL` | No | Only if LinkedIn blocks your IP |
| `LINKEDIN_PASSWORD` | No | Only if needed for access |

**Note:** LinkedIn has aggressive anti-scraping. If you get blocked, use LinkedIn's official API instead.

### JobStreet
| Field | Required | Notes |
|-------|----------|-------|
| `JOBSTREET_API_KEY` | No | Premium feature |
| `JOBSTREET_EMAIL` | No | Only for saved searches |
| `JOBSTREET_PASSWORD` | No | Only for saved searches |

### Indeed
| Field | Required | Notes |
|-------|----------|-------|
| `INDEED_API_KEY` | No | Indeed Publisher API for sponsored jobs |

**Get Indeed API Key:** https://www.indeed.com/publisher

### Glints
| Field | Required | Notes |
|-------|----------|-------|
| `GLINTS_API_KEY` | No | Basic scraping works without |

### Kalibrr
| Field | Required | Notes |
|-------|----------|-------|
| `KALibrr_API_KEY` | No | May need for some features |
| `KALIBRR_EMAIL` | No | For employer-matched jobs |
| `KALIBRR_PASSWORD` | No | For employer-matched jobs |

**Register:** https://www.kalibrr.id/developer

---

## 3. Base URL Configuration

For email feedback buttons to work, set your public URL:

```env
# For local testing:
BASE_URL=http://localhost:5000

# For VPS deployment:
BASE_URL=http://your-vps-ip:5000
# OR with domain:
BASE_URL=https://jobs.yourdomain.com
```

This URL is used to generate clickable feedback links in digest emails.

---

## 4. Database Credentials (Docker)

For local development, SQLite is used (no credentials needed).

For Docker/VPS deployment:

```env
DATABASE_URL=postgresql://jobuser:jobpass@db:5432/jobtracker
POSTGRES_USER=jobuser
POSTGRES_PASSWORD=your_secure_postgres_password
POSTGRES_DB=jobtracker
```

**Choose a strong password** - at least 16 characters, mix of letters/numbers/symbols.

---

## 5. Redis (Docker)

Default settings work for Docker Compose:

```env
REDIS_URL=redis://redis:6379/0
```

No authentication needed - Redis runs inside the Docker network only.

---

## 6. Flask Secret Key

Generate a random secret key for Flask sessions:

```env
SECRET_KEY=your-random-secret-key-here
```

**Generate a random key:**
```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

---

## Quick Start Checklist

- [ ] Gmail App Password configured
- [ ] EMAIL_USER and EMAIL_PASSWORD set
- [ ] BASE_URL set correctly (for your VPS or localhost)
- [ ] SECRET_KEY generated and set
- [ ] POSTGRES_PASSWORD changed from default

---

## Troubleshooting

### "Authentication failed" when sending email
- Wrong password - double-check your Gmail App Password
- 2FA not enabled on Google Account
- App Password not generated correctly

### Jobs not appearing from some portals
- Portal may be blocking your IP
- Try using VPN
- Add portal credentials if available

### Feedback links in email not working
- BASE_URL not set correctly
- Firewall blocking port 5000 (VPS)
- Web server not running
