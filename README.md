# Layered Risk-Based Conditional Access

Two Entra ID Conditional Access policies enforcing access decisions on independent Identity Protection risk signals — sign-in risk and user risk — with full remediation loop tested end to end.

## Environment

Hybrid Entra ID tenant `Google042.onmicrosoft.com` with on-prem AD `meridianiam.net` synced via Entra Connect. Entra ID P2 for Identity Protection and risk-based Conditional Access.

## Video Walkthrough

📺 Watch the walkthrough: https://youtu.be/6a1r6kqUeDc — full policy configuration, testing methodology, sign-in log analysis, and remediation loop demonstration.

## Problem

Identity Protection generates two distinct risk signals — sign-in risk (per-event) and user risk (cumulative). A single Conditional Access policy scoped to only one of these signals leaves half the attack surface unmonitored:

- **Sign-in-risk-only** — misses compromised users signing in from familiar locations
- **User-risk-only** — misses anomalous events targeting otherwise clean users
<img width="999" height="614" alt="image" src="https://github.com/user-attachments/assets/95b04506-09e6-4904-8e72-f0b3c5e5a728" />


Layering both is required for defense in depth in identity.

## Design

Two policies, each targeting one risk axis, each with a distinct grant control set aligned to that axis's threat model.

| Policy | Trigger | Grant Controls | Purpose |
|--------|---------|----------------|---------|
| `CA-Signin-Risk-High-MFA-Plus-Reauth` | Sign-in risk = High | MFA + Sign-in frequency Every time (5-min tolerance) | Catch anomalous individual sign-in events |
| `CA-User-Risk-High-MFA-Password-Change` | User risk = High | MFA + Require password change | Enforce remediation on confirmed-compromised users |

Both scoped to test user `achen@meridianiam.net`, targeting all cloud apps.

## Testing Methodology

Sign-in-risk policy tested by generating real Anonymous IP detections via Tor Browser sign-ins from rotating exit nodes (Dronten, Flevoland, NL).

<img width="741" height="318" alt="image" src="https://github.com/user-attachments/assets/7e481038-ebcb-4c2d-878e-43d90dca21d3" />

User-risk policy tested by confirming Alice compromised in Identity Protection and signing in from a routine familiar location, isolating user risk as the only elevated signal.

<img width="745" height="284" alt="image" src="https://github.com/user-attachments/assets/793bc55b-1ecb-4992-af2c-680f7092eaa7" />


## Results

**Sign-in-risk policy applied on Tor sign-ins:**
Sign-in log Conditional Access tab confirmed policy match, sign-in risk scored Medium, reauthentication prompts fired on token refresh to Outlook and SharePoint after 5-minute tolerance elapsed.

**User-risk policy applied on clean sign-in from confirmed-compromised user:**
Same test user signing in from home IP with sign-in risk scored None triggered the user-risk policy because user risk was High. 

<img width="958" height="105" alt="image" src="https://github.com/user-attachments/assets/c41bf0fb-42a2-48c8-9615-fc5edc89ac5e" />

<img width="759" height="233" alt="image" src="https://github.com/user-attachments/assets/109b4b5d-931e-4b1f-87a9-de663a7e0994" />

## Resolution

MFA required, password change forced, risk state remediated to normal on successful password change.

**Remediation loop closed correctly:**
Post-password-change verification confirmed user risk dropped from High to none, both policies stopped applying on subsequent sign-ins, and risk state moved to Remediated in Risky Users view.

## Key Findings

**Sign-in risk and user risk are independent axes.** Confirming a user compromised elevates user risk but does not re-score their future sign-ins as risky. A confirmed-compromised user signing in from a familiar location will score sign-in risk None. Policies must target both axes to enforce complete coverage.

**Target Resources must be explicitly configured.** Conditional Access fails silent when Target Resources is unset — the policy exists but attaches to no authentication event and never fires. Verification requires the Conditional Access tab on the sign-in log, not the policy Impact chart alone.

**"Sign-in frequency: Every time" enforces on token refresh, not wall-clock time.** The 5-minute tolerance permits reauthentication but does not schedule it. Reauthentication prompts fire only when the user's browser attempts to acquire or refresh a token past the tolerance window, typically on navigation to a new cloud resource.

**Most Conditional Access evaluations occur on non-interactive sign-ins.** Token refreshes for backend app calls are where most CA decisions get made. Analysts checking only the interactive sign-in log tab will miss the majority of policy enforcement activity.
