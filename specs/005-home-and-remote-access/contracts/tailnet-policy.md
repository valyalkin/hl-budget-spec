# Contract: Tailnet access policy

**Feature**: [../spec.md](../spec.md) | **Research**: R-006, R-007

The owner pastes this into the Tailscale admin console (*Access controls*), replacing the
default allow-all policy. `<owner-login>` (used only in `tests`) is the owner's Tailscale login:
`valyalkin@gmail.com`. Tailscale accepts comments and trailing commas (HuJSON).

```jsonc
{
  // Who may apply tag:homelab: the tailnet admin (the owner), and OAuth clients they create
  // with this tag. autogroup:admin keeps working if the owner's login changes.
  "tagOwners": {
    "tag:homelab": ["autogroup:admin"],
  },

  // The homelab gateway's single route is approved automatically.
  "autoApprovers": {
    "routes": {
      "10.0.0.100/32": ["tag:homelab"],
    },
  },

  // The owner's own devices may reach the homelab node on HTTP and HTTPS only.
  // Nothing lets tag:homelab start connections to the owner's devices.
  "grants": [
    {
      "src": ["autogroup:member"],
      "dst": ["10.0.0.100/32"],
      "ip":  ["tcp:80", "tcp:443"],
    },
  ],

  // Expected results; Tailscale refuses to save the policy if one fails.
  "tests": [
    {"src": "<owner-login>", "accept": ["10.0.0.100:443", "10.0.0.100:80"], "deny": ["10.0.0.100:22", "10.0.0.100:6443"]},
  ],
}
```

## Rules this policy guarantees

| Spec | How |
|---|---|
| FR-011: only the homelab address | The gateway advertises only `10.0.0.100/32`, and the grant allows only ports 80 and 443 |
| FR-012: only the owner's devices | `autogroup:member` is the owner's own signed-in devices (one-person tailnet); removing a machine ends its access |
| FR-013: no periodic sign-in | Tagged devices have key expiry disabled; the OAuth client secret does not expire |

## Settings outside the policy file

- **Device approval** (*Settings → Device management → Device approval*): optional. If it is on,
  the owner approves each new phone or laptop once.
- **Key expiry** for the owner's personal devices stays at the default (180 days). Re-signing in
  on a phone twice a year is acceptable; it can be disabled per device.
- **DNS**: the default settings are enough. Tailscale's resolver forwards to public DNS, which
  returns `10.0.0.100` for `*.<domain>`. This is also the fallback when a home router blocks
  private answers (research R-013).
