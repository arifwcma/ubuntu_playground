# Agent Direction — New Session Onboarding

Last updated: 2026-04-08

## Step 1 — Read preferences

Read `/home/ssm-user/pref.txt` before anything else. Follow all rules strictly.

Key rules from pref.txt:
1. Address user as Guru.
2. Start every response with a token: R1, R2, R3, etc.
3. Be absolutely brief. One sentence if possible.
4. One recommendation at a time. Ask before offering alternatives.
5. Never use em dashes.
6. Never include comments in code. If you do, Guru discards the entire response.
7. Numbered lists only. Never bullet points.
8. No extra context or theory unless asked.
9. Only provide code when asked or clearly required.
10. Use descriptive variable names.
11. Australian English spelling.

## Step 2 — Read the server log

Read `/home/ssm-user/sources/ubuntu_playground/log.md`. This is the full rebuild log for the server. It documents every step taken and all pending tasks.

## Step 3 — Periodic log maintenance

After any significant action in a session:
1. Update `/home/ssm-user/sources/ubuntu_playground/log.md` to reflect what was done.
2. Run: `cd /home/ssm-user/sources/ubuntu_playground && git add -A && git commit -m "update log" && git push`
3. Do this periodically during the session, not only at the end, in case the session closes abruptly.

## Step 4 — Understand the stack

- Server: AWS EC2 t3.medium, Ubuntu 24.04.4, IP 16.176.28.146.
- Access: AWS SSM Session Manager only. No SSH.
- Apps run in Docker. One shared network: internal-apps.
- Nginx is sole public entry point (ports 80, 443).
- wcma.work → mapnj2 (Next.js app)
- testpozi.online → QGIS Server
- Full directory structure and configs are in log.md.

## Step 5 — Check pending tasks

Section 13 of log.md lists all pending tasks. Ask Guru which to work on before starting anything.
