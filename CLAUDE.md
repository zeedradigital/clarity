# Clarity — handoff for Claude

Personal expense tracker for Faiz. One-file app: `index.html` (all HTML/CSS/JS) + `help.html` (user guide — keep it in step with features).
Live at **https://clarity.zeedra.in**. Repo: https://github.com/zeedradigital/clarity (PUBLIC — never commit personal amounts, keys beyond the existing Supabase anon key, or backups).

## Going live
- Vercel project `gap-dashboard` (team `team_cWGww3OoXzOKOZI1WteDadPw`, project `prj_3fIlXzbsjTTO6yjYK6xBcQibQgA8`).
- **Vercel is NOT linked to GitHub** — pushing does not deploy. Faiz deploys by running: `! cd /c/landing-pages/clarity && vercel --prod --yes`
- After a deploy, verify on the live tab (e.g. `typeof someNewFunction`) and have Faiz refresh the phone (close + reopen).
- `.vercelignore` keeps `docs/`, `testing/`, `CLAUDE.md` off the website.

## Data
- State object `S` → localStorage `gd_v4_<userId>` and Supabase table `user_state` (`user_id`, `data`, `updated_at`), via `save()` → `scheduleSync()` → `pushRemote()`; `pullRemote()` → `mergeState()` on login.
- Entry: `{id,date,type:'income'|'expense'|'repayment',amount,note,cat,fund:'own'|'credit',rotating,loanId,loanName,investCleared,_m}`.
- **Sync rule:** same entry on two devices → higher `_m` (edit time) wins; deleted ids live in `S.deletedIds` and never come back. Any code or console fix that changes an existing entry **must set `t._m=Date.now()`**, or a stale device will undo it (this happened once on 28 Oct 2026 — the phone reverted 52 entries).
- **Tags only (no categories in the UI).** The tag is the entry's `note`. Each tag has one type in `S.tagNature[key]`: `mean` Essential · `ind` Non-essential · `inv` Invest · `none` Don't count. `natureOf(note,cat)` reads `tagNature` first, then falls back to the hidden old `cat`. `cat` still exists on entries — don't remove it.
- Vague tag names are blocked (`VAGUE_TAGS`: essential, personal, family, business, invest, other…). `food` is a real tag — not blocked.
- Debts: `S.loans` `[{id,name,emi,balance}]`; `applyLoans()` sets `settings.totalDebt`/`monthlyFixed`.
- Balance: `ownMoneyLeft()` = `balanceFixes` + income − own spends − own repayments. Faiz sets the real balance with "Correct balance" — never overwrite `balanceFixes`.
- Wealth meter: `duelFor(y,m)` → repay/invest vs essential/non-essential.

## Faiz's decisions (don't undo)
- Keep all old records — never delete history.
- Meta ads = Office/business spend (Essential), not Invest.
- Bank interest, e-NACH charges, "Repayment — interest" = Don't count.
- Tags renamed 28 Oct 2026: essential→Daily needs, family→Family expense, personal→Personal spend, business→Office, friend→Friends, foo→food.

## How to work on it
- Edit with small scripted replacements (assert each match once), then syntax-check: extract `<script>` blocks → `node --check`.
- Test on an offline copy before going live: copy `index.html` to the scratchpad **without** the `supabase-js@2` script line, serve with `python -m http.server 8765 --bind 127.0.0.1`, open in Chrome, call `onLogin({user:{id:'t1'}})`, seed `S.txs`, override `confirm`/`prompt`. Clear localStorage and stop the server afterwards.
- Changing live data: Faiz must be logged in on the live tab; take a backup first (`localStorage.setItem('clarity_backup_<date>_<what>', JSON.stringify(S))`), compare meters/balance before and after, set `_m`, then check the Supabase copy.
- Commit + push to GitHub after each change (backup only).

## Open items (Faiz to decide)
- Some spends have no tag — Settings → Tags shows how many; tag them in History.
- "Rayyan" spend is Don't count — change if it should count.
- `food` is Essential — may be Non-essential.
- Debt names "Card 1–6" could be renamed; EMIs not entered yet.
- Repayment chips have no long-press (types don't apply to repayments).
