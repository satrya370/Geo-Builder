# CP 019 — Authority/Freshness/Citation Revision Log

## §1 — WordPress JSON-LD Survival Verification

Status: in-progress — §1 verified; §2/§3 ready to implement.

- Read `ai_docs/019_geo-authority-freshness-revision_plan.md` before workflow changes.
- Attempted to open WordPress Admin for existing draft post ID `10` at `wordpress.com/wp-admin/post.php?post=10&action=edit`.
- WordPress redirected to login and returned HTTP `403` (`Checking your browser...`).
- User published post ID `10` manually at `https://satryapudja.wordpress.com/2026/07/28/stoic-ethics-mengapa-kebajikan-penting/`.
- Public HTML fetch returned HTTP `200` and 122,502 bytes.
- `application/ld+json`: absent.
- `BlogPosting`: absent.
- `FAQPage`: absent.
- `datePublished` and the tested visible date patterns: absent.
- Author-related content was present, but this does not prove JSON-LD survival.
- Conclusion: WordPress.com strips the workflow's `<script type="application/ld+json">` from the public page.
- No microdata/RDFa workaround was implemented; that option requires separate Reviewer discussion as specified by the plan.

### §1 Evidence

- Public URL was provided by the user and fetched directly, not through the REST API.
- The result determines that schema markup cannot be relied on as a WordPress.com on-page signal.
