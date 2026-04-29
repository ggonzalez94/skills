---
name: gh-image-video
description: Use whenever the user wants to attach an image, video, screenshot, screen recording, GIF, PDF, or any other binary file to a GitHub PR (description, comment, or review) without committing it to the repo. Triggers on phrases like "attach this screenshot to the PR", "add a recording to the PR description", "embed this video in the PR", "host this image for the PR", "upload this file for the PR", or any flow that produces a PR-ready link to a non-source asset. Uploads to a public Google Cloud Storage bucket and prints a markdown-ready link. Use this even when the user doesn't explicitly say "upload" — if the end goal is "show this in a PR" and the file isn't already hosted somewhere, this is the skill.
---

# gh-image-video

Host images, videos, and other binary attachments on a public GCS bucket so they can be embedded in GitHub PRs without bloating the repo.

GitHub renders images and videos inline if the URL is publicly fetchable. Committing them to the repo balloons history and is rarely what the user wants. This skill puts them on `gs://gh-evidence` and gives the user a markdown snippet they can paste straight into a PR.

## Backend

Google Cloud Storage only, for now. Bucket: `gh-evidence`. Public read at the bucket level (uniform bucket-level access + `roles/storage.objectViewer` granted to `allUsers`).

If the user asks for a different backend (S3, R2, Imgur, etc.), say it's not supported yet — don't improvise.

## Workflow

### 1. Sensitivity check — before any upload

The bucket is **public**. Anything uploaded is reachable by anyone with the URL and may be indexed.

Before uploading, look at what's about to go up:

- If it's an image you can preview, look at it. Flag anything that looks like a `.env`, a dashboard with PII, an internal admin panel, a Slack thread with names, customer data, API keys in a terminal, etc.
- If it's a non-image (`.csv`, `.zip`, `.log`, `.pdf`), you can't easily inspect it. Ask the user to confirm there's nothing sensitive inside.
- If the filename or path contains `secret`, `credential`, `private`, `internal`, `confidential`, `.env`, `.key`, `.pem`, `.p12`, `.kdbx`, or matches a database dump pattern (`*.sql`, `pg_dump.*`), stop and ask for explicit confirmation.

For obviously innocuous uploads (a screenshot of a public website, an animation of an OSS project), one quick "this is going to a public URL — OK?" is enough. Don't pile on warnings.

If the user has already given consent earlier in the session for a similar file, you don't need to re-ask for each one. Use judgment.

### 2. Preflight

Run these in order. Bail at the first failure with the suggested message — don't try to work around problems silently.

**a. gcloud installed**

```bash
command -v gcloud
```

If missing, tell the user:
> `gcloud` isn't installed. Install the Google Cloud SDK from https://cloud.google.com/sdk/docs/install and re-run.

Don't try `gsutil`, `aws`, `curl` to a different host, or anything else.

**b. Logged in**

```bash
gcloud auth list --filter=status:ACTIVE --format="value(account)"
```

If empty, tell the user:
> No active gcloud account. Run `gcloud auth login` and re-run.

**c. Bucket reachable**

```bash
gcloud storage buckets describe gs://gh-evidence --format="value(name)"
```

If this fails with `NotFound`, the bucket doesn't exist on the active account's projects. If it fails with a permission error, the active account can't see it (could be the wrong account, or the bucket lives elsewhere).

In either case, surface what failed and offer to create it. Don't auto-create — get explicit confirmation. See "Bucket setup" below.

### 3. Upload

Pick a destination path that won't collide with previous uploads:

```
<YYYYMMDD-HHMMSS>-<short-random>-<sanitized-original-filename>
```

Example: `20260429-143012-a3f9-screenshot.png`

Use timestamp (in the user's local time is fine), 4 hex chars of randomness, and the original basename with spaces replaced by `-`. This keeps URLs readable and avoids clobbering.

Upload:

```bash
gcloud storage cp <local-path> gs://gh-evidence/<destination-path>
```

Don't pass `--predefined-acl=publicRead` — the bucket uses uniform bucket-level access and per-object ACLs are disabled. The IAM binding handles public read.

### 4. Output a PR-ready snippet

Public URL is always:

```
https://storage.googleapis.com/gh-evidence/<destination-path>
```

Format the snippet based on file type:

| File type | Snippet |
|---|---|
| Image (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg`) | `![<original-filename>](<url>)` |
| Video (`.mp4`, `.mov`, `.webm`) | Just the bare URL on its own line — GitHub auto-embeds a player for these |
| Anything else | `[<original-filename>](<url>)` |

Print both the URL and the snippet so the user can copy whichever they need.

If the user is mid-flow on a PR (e.g., this came up while running `gh pr create` or editing a PR body), offer to inject the snippet into the description directly.

## Bucket setup (first run only)

Only run this if the preflight reported the bucket doesn't exist and the user said yes to creating it.

You'll need a GCP project. Check with:

```bash
gcloud config get-value project
```

If empty, ask the user which project to use. List options with `gcloud projects list`. They can create one with `gcloud projects create <id>` if needed.

If the user has multiple gcloud accounts, ask which one should own the bucket — the bucket lives under one account's project, and that's where all uploads will count toward billing. Set the active account with `gcloud config set account <email>` before creating.

Create:

```bash
gcloud storage buckets create gs://gh-evidence \
  --location=US \
  --uniform-bucket-level-access \
  --project=<project-id>
```

Make objects publicly readable:

```bash
gcloud storage buckets add-iam-policy-binding gs://gh-evidence \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

If `add-iam-policy-binding` fails because of an org policy that blocks public buckets (`constraints/iam.allowedPolicyMemberDomains`), tell the user — they can't host public assets on that account, and they'll need a different GCP account or a different storage backend (which this skill doesn't support yet).

Verify before claiming success:

```bash
gcloud storage buckets describe gs://gh-evidence --format="value(name,iamConfiguration.uniformBucketLevelAccess.enabled)"
```

## Why these defaults

- **`gh-evidence` as a fixed bucket name** — keeps URLs predictable, lets the user paste links into PRs without thinking about which bucket.
- **Uniform bucket-level access + IAM, not per-object ACLs** — per-object ACLs are the legacy approach and Google recommends against them. Uniform access is simpler, faster, and the supported path going forward.
- **Public bucket, not signed URLs** — PR reviewers shouldn't have to deal with expiring links. The whole point is GitHub renders the image inline forever.
- **Timestamped + hashed filenames** — collisions would silently overwrite earlier evidence; that's worse than verbose URLs.
- **Sensitivity check first** — once a file is on a public URL, you can delete the object but you can't un-publish what may have been crawled. The 5-second confirmation is the cheapest safeguard available.
