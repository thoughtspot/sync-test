# PR Merge Webhook Notifier

This repository contains a GitHub Actions workflow that automatically sends webhook notifications when Pull Requests are merged. The workflow captures relevant PR information, computes a secure signature for authentication, and delivers the data to your specified webhook endpoint.

## Features

- Triggers on any PR merge (to any branch)
- Includes comprehensive PR and repository information in the payload
- Implements HMAC-SHA256 signature-based authentication
- Includes timestamp verification to prevent replay attacks
- Debug mode for testing without sending actual webhook requests

## Setup Instructions

### 1. Add the Workflow File

Create a directory structure `.github/workflows/` in your repository if it doesn't already exist, then add the workflow file `pr-merge-webhook.yml` there.

### 2. Configure GitHub Secrets

Go to your repository settings:

1. Navigate to Settings → Secrets and variables → Actions
2. Add the following repository secrets:
   - `WEBHOOK_URL`: The URL of your webhook endpoint
   - `WEBHOOK_SECRET`: A secure random string used for signature generation
   - `SEND_WEBHOOK`: Set to "false" for testing, "true" for production use

### 3. Testing

To verify everything works correctly:

1. Initially leave `SEND_WEBHOOK` set to "false" (or don't set it at all)
2. Create and merge a test PR
3. Check the GitHub Actions logs for the debug output including the payload and headers
4. Verify the payload structure contains all required information
5. When ready, set `SEND_WEBHOOK` to "true" to enable actual webhook delivery

## Webhook Payload

The webhook delivers a JSON payload with the following structure:

```json
{
  "repository": {
    "name": "repo-name",
    "full_name": "owner/repo-name",
    "owner": "owner"
  },
  "pull_request": {
    "number": 123,
    "title": "PR title",
    "merged_by": "username",
    "merged_at": "2023-01-01T12:00:00Z",
    "html_url": "https://github.com/owner/repo-name/pull/123",
    "base": {
      "ref": "main",
      "sha": "base-commit-sha"
    },
    "head": {
      "ref": "feature-branch",
      "sha": "head-commit-sha"
    }
  }
}
```

## Security Implementation

The webhook implementation uses HMAC-SHA256 signatures for authentication:

1. Each request includes the following headers:
   - `X-Hub-Signature-256`: HMAC signature using SHA-256
   - `X-Webhook-Timestamp`: UTC timestamp when the request was generated

2. The signature is computed from both the timestamp and the payload:
   ```
   HMAC-SHA256(secret, timestamp + payload)
   ```

3. Your webhook receiver should:
   - Extract the signature and timestamp from headers
   - Compute the expected signature using the same secret
   - Compare signatures to verify authenticity
   - Optionally verify the timestamp is recent to prevent replay attacks

## Webhook Receiver Implementation

When implementing your webhook receiver, use the following verification process:

```go
// Example signature verification in Go
func verifySignature(secret, payload []byte, timestamp, signature string) bool {
    // Create HMAC hasher
    h := hmac.New(sha256.New, []byte(secret))
    
    // Write timestamp and payload
    h.Write([]byte(timestamp))
    h.Write(payload)
    
    // Get computed signature
    computedSignature := fmt.Sprintf("%x", h.Sum(nil))
    
    // Compare signatures using constant-time comparison
    return hmac.Equal([]byte(signature), []byte(computedSignature))
}
```

## Customization

To customize the payload, modify the JSON structure in the workflow file. GitHub context variables provide access to extensive information about the repository and PR that you can include as needed.

## Troubleshooting

- **Workflow doesn't trigger**: Ensure the PR is being merged, not just closed
- **Payload issues**: Check the debug output in GitHub Actions logs
- **Webhook not receiving**: Verify the URL is correct and accessible from GitHub's servers
- **Signature verification fails**: Ensure the same secret is used on both sides
