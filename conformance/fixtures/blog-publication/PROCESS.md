---
name: blog-publication
description: Draft a blog post, obtain editor approval and publish the approved revision at the scheduled time.
inputs:
  brief: string
  slug: string
  publishAt: datetime
steps:
  - id: draft
    agent: |
      Draft the post from the brief in the CMS, checking sources and permissions
      for images. Keep it unpublished. Address the editor's rejection note on
      revisions. Attach an immutable preview of this revision.
    output:
      revision: string
      title: string
      summary: string
    evidence: [link]
  - id: editor_review
    person: editor
    approve: Approve the exact draft revision for accuracy, tone, sources and image rights.
    on_reject: draft
    due: 2d
  - id: publication_time
    wait_until: inputs.publishAt
  - id: publish
    agent: |
      Publish only the approved draft revision using slug. If the CMS revision
      has changed, escalate rather than publishing. Check whether that revision
      is already live before writing. Verify that the public URL is accessible.
    output:
      url: string
      publishedRevision: string
    evidence: [link]
  - id: done
    finish: published
---

# Blog publication

Marketing supplies an absolute publishAt with a timezone. If approval comes
after publishAt, publish as soon as approved. The editor reviews the stored
revision, not a mutable latest-draft page. A schedule change requires cancelling
this run and starting another; this process does not edit its inputs.
