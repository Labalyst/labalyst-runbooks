# Test file — DO NOT MERGE

This file exists solely to verify that the secret-scan CI gate
correctly blocks PRs that contain credential-shaped strings.

The next line contains a deliberately-fake Clerk secret-key-shaped
string that should match the `clerk-secret-key` rule in
`.gitleaks.toml`:

CLERK_SECRET_KEY=sk_test_AAAAAAAAAAAAAAAAAAAAAAAAAAAA

If gitleaks is configured correctly, the PR opened from this branch
will fail the `secret-scan / gitleaks` status check, and branch
protection will block merging into main. After verifying, this branch
should be deleted without merging.
