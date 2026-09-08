# Fork notes

This repository is a downstream of [elie222/inbox-zero](https://github.com/elie222/inbox-zero)
that carries a small patch series and publishes its own image. The deployment
that consumes it lives in [afonsojramos/cloud](https://github.com/afonsojramos/cloud)
(`hetzner/stacks/inbox-zero/docker-compose.yml`).

## Branches

| Branch   | Contents                                                       |
| -------- | -------------------------------------------------------------- |
| `main`   | Exact mirror of upstream `main`. Never commit here.            |
| `custom` | Upstream `main` plus the patch series. Default branch; deployed. |

The patch series is the set of commits between the two branches:

```sh
git log --oneline main..custom          # list the patches
git format-patch main..custom -o patches # export them as .patch files
```

Every fork commit is one self-contained patch with a descriptive message, so
the series stays readable and rebases cleanly. Do not merge `main` into
`custom`; the sync workflow rebases instead so the series stays linear.

## Current patches

- **feat(filing): let the AI create nested folders inside allowed filing folders**
  Document filing used to create every new folder at the drive root and the AI
  had no way to say "inside this folder". The analysis schema gains
  `parentFolderId`, new paths are resolved against the user's allowed folders
  (a path that starts with a known folder's path nests beneath it), the filing
  prompt tells the model it owns the folder structure inside those folders, and
  reply-driven moves resolve paths the same way.
- **chore(fork): add upstream sync and image publish workflows**
  The two `fork-*` workflows below and this file.

## Staying current with upstream

`.github/workflows/fork-upstream-sync.yml` runs every Monday and on demand:

1. fetches upstream `main` and force-mirrors it into `main`;
2. rebases `custom` onto it and pushes with `--force-with-lease`;
3. publishes a new image through `fork-publish-image.yml`;
4. disables upstream's scheduled workflows (they need upstream secrets) via the
   API instead of editing their files, which would conflict on every rebase.

When the rebase conflicts nothing is pushed. The workflow opens (or comments on)
an issue titled "Upstream sync conflict" listing the files. Resolve it locally:

```sh
git fetch origin custom main
git checkout -B custom origin/custom
git rebase origin/main
# fix conflicts, git add, git rebase --continue
pnpm -F inbox-zero-ai exec vitest --run utils/drive utils/ai/document-filing
git push --force-with-lease origin custom
```

Pushing `custom` publishes a new image; the issue closes on the next clean run.

## Image

`.github/workflows/fork-publish-image.yml` builds `docker/Dockerfile.prod` for
`linux/amd64` on every push to `custom`, after running the tests that cover the
patches, and pushes `ghcr.io/afonsojramos/inbox-zero` with three tags:

| Tag                   | Meaning                                                     |
| --------------------- | ----------------------------------------------------------- |
| `<YYYYMMDD>.<HHMMSS>` | Committer date (UTC) of the built commit. Pinned by cloud.  |
| `sha-<short sha>`     | Exact commit.                                               |
| `custom`              | Latest build of the branch. Never deploy this moving tag.   |

The dated tag is what the cloud repository pins; Renovate reads it with a regex
versioning so newer builds show up as ordinary dependency PRs there. Merging
that PR is the deployment step, so a bad upstream change never reaches the
server without a review.

The GHCR package must be public for the server to pull it without credentials.
GitHub creates it private on the first push: open the package settings under
the repository's **Packages** and change its visibility once.

## Adding a patch

1. Branch from `custom`, make the change with tests, and keep it to one commit
   per concern with a conventional-commit subject.
2. Add the test path to the `verify` job in `fork-publish-image.yml` if the
   patch is not already covered by `utils/drive` or `utils/ai/document-filing`.
3. Describe the patch in **Current patches** above.
4. Push to `custom` (or open a PR against it). The image publishes on push.

Consider sending patches upstream too; the fewer commits the series carries,
the fewer rebases conflict.
