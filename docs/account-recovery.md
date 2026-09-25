# Account Recovery Guide

This guide helps users recover from common wallet and network connection issues when using Xconfess.

## Problem

You may have connected the wrong wallet or the wrong network to Xconfess. This can cause:
- Inability to access your account
- Missing data or balances
- Transaction failures
- Login loops or unexpected errors

## Recovery Steps

### Wrong Wallet Connected

1. Click the wallet icon in the top right corner of the interface
2. Click **"Disconnect"** to disconnect the current wallet
3. Click **"Connect Wallet"**
4. Select the correct wallet from the list
5. Approve the connection request in your wallet extension

### Wrong Network Connected

1. Open your wallet extension (e.g., MetaMask, Phantom, etc.)
2. Switch to the correct network (e.g., Ethereum Mainnet, BSC Mainnet, testnet, etc.)
3. Refresh the Xconfess page
4. Your data should now appear correctly

### Both Wallet and Network Are Wrong

1. Disconnect the current wallet (see steps above)
2. Connect the correct wallet
3. Ensure the wallet is on the correct network
4. Refresh the page
5. Verify your account data loads properly

### Still Having Issues?

- Clear your browser cache and cookies
- Try a different browser or private/incognito window
- Restart your wallet extension
- Ensure your wallet is unlocked
- Check that your wallet supports the required network
- Contact support if the problem persists

## Common Failure Modes

- **Network mismatch**: You're on testnet but the app expects mainnet (or vice versa)
- **Wallet not supported**: The wallet you're using isn't supported by Xconfess
- **Session expired**: Your session has timed out and requires re-authentication
- **Browser extension conflict**: Multiple wallet extensions are interfering with each other
- **Cached connection**: The app is reading a stale connection from local storage
- **Account lock**: Too many failed attempts temporarily locked the account

## Prevention

To avoid these issues in the future:

- **Always double-check the network** before connecting your wallet
- **Use the same wallet consistently** to avoid data fragmentation
- **Keep your wallet software updated** to the latest version
- **Disconnect when not actively using** the application
- **Bookmark the official Xconfess site** to avoid phishing sites
- **Verify the network icon** in your wallet before approving transactions
- **Clear cache periodically** if you experience strange behavior

## Login Anomaly Detection and Step-Up Challenges

Xconfess continuously evaluates login attempts for signs of credential stuffing and impossible-travel patterns. When an attempt looks suspicious, the user is asked for a second factor (a step-up challenge) or the attempt is blocked, depending on the computed risk.

### Anomaly Signals

Each login attempt is scored using the following signals. Signals are derived from the attempt itself and from recent history; raw sensitive values are never stored.

- **Credential stuffing**: Many distinct accounts are attempted from the same source in a short window, or a single account is attempted from many sources.
- **Impossible travel**: Two successful or attempted logins for the same account originate from locations that cannot be reached in the elapsed time.
- **Velocity**: The number of attempts for an account or source exceeds a configured threshold within a rolling window.
- **Device novelty**: The attempt comes from a device fingerprint not previously associated with the account.
- **IP novelty**: The attempt comes from an IP address or network not previously associated with the account.

### Risk Scoring

Signals are combined into a single risk score per attempt. Each signal contributes a weighted amount, and the total maps to a risk band:

- **Low**: The attempt proceeds normally.
- **Medium**: The attempt proceeds but is flagged for operator review and contributes to future scoring.
- **High**: The attempt requires a step-up challenge (a second factor) before it can proceed.
- **Critical**: The attempt is blocked outright and the account owner is notified.

Thresholds and weights are configurable per deployment so operators can tune sensitivity without code changes.

### Step-Up Behavior

- **High risk**: The user must complete a second factor (for example, a one-time code or an additional wallet signature) before the session is established. If the challenge fails or expires, the attempt is treated as blocked.
- **Critical risk**: The attempt is rejected immediately. The user is shown a clear message and, where appropriate, a recovery path.
- **Medium risk**: The attempt succeeds, but the event is recorded for review and raises the account's baseline risk for subsequent attempts.

### Correlation by Request ID

Every login attempt, anomaly signal, risk decision, and step-up challenge is correlated by a single request ID. This ID is attached to all related events so that operators can trace a full attempt end to end, and so that support can reference a specific attempt without exposing sensitive data.

### Privacy Limits

Anomaly detection is designed to minimize the data it retains:

- **Data minimization**: Only the signals needed for scoring are derived. Raw credentials, full IP addresses, and precise locations are not stored; they are hashed or truncated where a stable identifier is required.
- **Retention**: Derived signals and risk events are retained only for the configured retention window, after which they are deleted or aggregated.
- **No raw sensitive data leakage**: Logs and operator views never contain raw credentials, full IP addresses, or precise location data. Events reference the request ID and the derived signal values only.
- **False-positive measurement**: Risk decisions are recorded with their outcome so that false positives can be measured and thresholds tuned over time.

### If You Are Challenged or Blocked

If a login is challenged or blocked, you can recover as follows:

1. Complete the step-up challenge if one is presented.
2. If the attempt was blocked, wait for the configured cooldown and try again from a network and device you normally use.
3. If you believe the block is a false positive, contact support and reference the request ID shown in the error message.
4. If you suspect your account is under attack, change your credentials and review connected wallets.

## Account Deletion Orchestration

Account deletion is a stateful, multi-step process. It spans posts, messages, exports, notifications, analytics, and chain references, each with different retention requirements. The orchestration job tracks an explicit state so that deletion is idempotent and observable, and so that user-facing status is always accurate.

### Deletion States

A deletion request moves through the following states:

- **requested**: The user has asked to delete their account. No data has been removed yet.
- **confirmed**: The user has explicitly confirmed the deletion request.
- **grace_period**: A configurable waiting window during which the user can cancel the deletion.
- **processing**: The orchestration job is actively deleting or anonymizing records across all subsystems.
- **completed**: All deletable records have been removed or anonymized and the account is closed.
- **failed**: The job encountered an error and stopped. The job can be retried safely.

### Confirmation

Deletion never proceeds without explicit user confirmation. A request in the `requested` state does not remove any data. Only after the user confirms does the job transition to `confirmed` and then to `grace_period`.

### Grace Period

After confirmation, the job enters a configurable grace period. During this window the user can cancel the deletion, which returns the account to normal operation. The grace period length is configurable per deployment so that operators can tune it to their retention and support policies.

### Anonymization Rules

Records that must be retained for legal or operational reasons are de-identified rather than deleted. Anonymization replaces direct identifiers (such as wallet address, display name, and contact details) with non-reversible placeholders while preserving the record's structural integrity for analytics and audit purposes.

### Legal-Retention Exceptions

Some records are subject to legal-retention requirements and cannot be deleted within the normal flow. These records are:

- Retained for the minimum period required by applicable law.
- De-identified using the anonymization rules above.
- Excluded from the user-facing deletion confirmation until the retention period expires.

Retained records are always justified by a documented retention requirement and are never left in an identifiable form.

### Idempotency and Observability

Each transition is idempotent: re-running the job in any state produces the same result and does not duplicate work. The job emits status updates so that the user-facing status reflects the true state of the deletion at all times.

### Cancelling a Deletion

If you requested deletion by mistake, you can cancel it during the grace period:

1. Open your account settings
2. Locate the pending deletion notice
3. Click **"Cancel Deletion"**
4. Confirm the cancellation

Once the job has entered `processing`, cancellation is no longer possible.
