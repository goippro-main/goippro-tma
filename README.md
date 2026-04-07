# GoIPpro Telegram Mini App

Partner-facing Telegram Mini App for [GoIPpro](https://goippro.com) — earn money by routing incoming calls through your SIM cards + GoIP device.

## Features

- 7 screens: Onboarding, Calculator, Registration, Confirmation, Dashboard, Referrals, Support
- 6 languages auto-detected from Telegram: **EN, FR, AR (RTL), ES, PT, ID**
- Connects to GoIPpro backend API (`.194`)
- Telegram WebApp SDK: haptic feedback, theme detection, MainButton, openLink
- Mobile-first design (320–480px), teal `#0D9488`, dark theme

## Screens

| Screen | Description |
|--------|-------------|
| 🏠 Home | Onboarding with country flag, key benefits |
| 💰 Earn | Earnings calculator (23 countries, 4/8/16/32 SIM slots) |
| 📝 Register | Registration → `POST /auth/register-partner` |
| ✅ Confirm | Success screen with next steps |
| 📊 Dashboard | Partner stats via `GET /partner/me` |
| 🔗 Refer | Referral link generation & sharing |
| 💬 Support | FAQ accordion + @goippro_support |

## Stack

- Pure HTML/CSS/JS (no build step, instant deploy)
- [Telegram WebApp SDK](https://core.telegram.org/bots/webapps)
- Backend: `https://api.goippro.com` (Laravel/Sanctum, `.194`)

## Deploy

### Hostinger (production)
FTP to `goippro.com/tma/` — see CI/CD docs.

### Telegram Registration
1. Create bot via @BotFather → `/newapp`
2. Set URL: `https://tma.goippro.com` (or `https://goippro.com/tma/`)
3. Link: `https://t.me/goippro_bot/app`

## API Endpoints Used

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/cc/api/auth/register-partner` | New partner registration |
| POST | `/cc/api/auth/login` | Login |
| GET | `/cc/api/partner/me` | Partner profile |

## TODO

- [ ] Bot token from BotFather → register TMA
- [ ] DNS: `tma.goippro.com → 195` + SSL
- [ ] Backend: add `/partner/stats`, `/partner/payout` endpoints
- [ ] Telegram auth via `initData` HMAC verification
- [ ] Push notifications when payout is ready

## Changelog

### v0.1.0 — 2026-04-08
- Initial release: all 7 screens
- 6 language translations (EN/FR/AR/ES/PT/ID)
- RTL support for Arabic
- Calculator: 23 countries, 4 slot options
- Auth: register + login connected to real API
- Dashboard: live data from `/partner/me`
- Referral link generator
- FAQ accordion (5 Q&A per language)
