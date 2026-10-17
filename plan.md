# Self-contained Git bundle creation

## Goal

Add a public API for creating self-contained local Git bundles. A caller should
be able to select references, write a bundle, and use that bundle with Git or
libgit2 to clone or fetch into a repository with no prerequisite objects.

This follows the read-only bundle support merged in #7338. Develop against
`main`; the fuzz coverage in #7433 is independent and does not need to merge
first. Keep parser behavior changes out of this work unless a concrete defect
blocks creation or interoperability.

## Prior work and review lessons

- `origin/git-bundles` (7e8a8d0e6, from #7221) contains an earlier
  `git_bundle_create` and `tests/libgit2/bundle/create.c`. Mine its test cases;
  do not reuse its design (`bundle_version` option, opaque `git_bundle` handle,
  `bundle://` URL prefix), which was dropped or criticized in review.
- #7101 (Laurence McGlashan) is the original proposal. On #7221 lrm29 asked
  that follow-up work build visibly on earlier work; credit the lineage in the
  PR description as #7338 did.
- The #7338 review consistently favored fewer states (an enum probe became a
  `bool`), no heavy caller-owned objects (reader and header folded into the
  parser lifecycle), overridable dispatch, and short, accurate comments. Apply
  the same bias here: one-shot API, no builder, no speculative options, and no
  comments that justify more than the code does.

## Scope

- Create self-contained bundles containing explicitly selected references and
  all objects reachable from their advertised OIDs.
- Support SHA-1 and, in builds that support it, SHA-256 repositories.
- Preserve annotated tag objects and their targets.
- Support progress reporting and cancellation using existing conventions.
- Stream the pack to the output without buffering the entire bundle in memory.
- Provide deterministic reference ordering; do not promise byte-identical packs.

Defer incremental bundles and prerequisites, revision-expression parsing,
filtered bundles, shallow repository support, bundle-URI discovery or download,
in-memory or stream output variants, and changes to clone's handling of missing
`HEAD`. Bundle creation is a separate operation from pushing to a bundle
transport.

## 1. Settle the public API contract

Start with a small proposed API in `include/git2/bundle.h`, with documentation
and a usage example, before implementing the writer. Wire the header into
`include/git2.h`. Roughly:

```c
typedef struct {
	unsigned int version;
	git_packbuilder_progress progress_cb;
	void *progress_cb_payload;
} git_bundle_create_options;

#define GIT_BUNDLE_CREATE_OPTIONS_VERSION 1
#define GIT_BUNDLE_CREATE_OPTIONS_INIT { GIT_BUNDLE_CREATE_OPTIONS_VERSION }

GIT_EXTERN(int) git_bundle_create_options_init(
	git_bundle_create_options *opts, unsigned int version);

GIT_EXTERN(int) git_bundle_create(
	git_repository *repo,
	const char *path,
	const git_strarray *refs,
	const git_bundle_create_options *opts);
```

Follow the options conventions exactly, including the
`@options[version|init_macro|init_function]` doc tags (see
`include/git2/worktree.h`). Reuse the `git_packbuilder_progress` typedef rather
than inventing a callback type. Names remain open to maintainer feedback; the
shape (one-shot, file destination, versioned options) should not need to.

Specify these behaviors explicitly:

- Accept only full reference names (`refs/...`) and `HEAD`; shorthand such as
  `main` is rejected, since revision parsing is out of scope.
- Resolve and snapshot selected refs and OIDs once. Resolve symbolic refs;
  reject missing, broken, or unborn selections before creating any output.
- Require explicit selection of `HEAD` when callers want it advertised. Do not
  invent a `HEAD` entry or advertise refs that were not selected.
- Write header entries in caller order, dropping later duplicates. Do not
  special-case `HEAD`: the reader already advertises it first
  (`move_head_first` in `src/libgit2/transports/bundle.c`). Caller order is
  this API's contract; matching Git's header ordering is not required.
- Include annotated tags using their unpeeled OIDs.
- Reject an empty selection. Decide and test whether valid refs to trees or
  blobs are supported; prefer supporting them if consistent with Git's format
  and the existing reader.
- Default to v2 for SHA-1 and v3 with `object-format=sha256` for SHA-256.
  Do not expose a format-version option without a concrete use case.
- Reject shallow sources (`git_repository_is_shallow`). Missing reachable
  objects in partial or otherwise incomplete repositories must cause creation
  to fail.
- Document callback lifetime, cancellation/error propagation, file mode, and
  replacement semantics as provided by `git_filebuf` (section 3).

Exit criterion: the API and its edge cases are concrete enough to review without
having to infer behavior from the implementation.

## 2. Collect the complete object closure

Use the existing packbuilder and revision walker, following the pattern in
`local_download_pack` (`src/libgit2/transports/local.c`): push commits to a
revwalk, `git_packbuilder_insert_walk`, then `git_packbuilder_foreach`.

Do not copy that function's handling of non-commit tips. It passes tags to
`git_packbuilder_insert_recur`, which inserts a tagged commit with
`git_packbuilder_insert_commit`: that commit and its tree only, not its parents
(`src/libgit2/pack-objects.c`). Local fetch gets away with this because tagged
commits are usually also reachable from a branch; a bundle selecting only a tag
would be missing history. Instead, for each snapshotted tip:

- Peel through the tag chain, `git_packbuilder_insert` each tag object.
- Push a peeled commit to the revwalk.
- Insert a peeled tree or blob with `git_packbuilder_insert_recur`.

Overlapping histories are deduplicated by the packbuilder. Tree insertion
already skips gitlink entries, so submodule commits are not traversed. The
resulting pack must require no external delta bases or prerequisite objects.

Exit criterion: tests establish complete closure for multiple branches, merged
and disconnected histories, a tag-only selection whose commit has parents,
tag chains, and supported non-commit tips.

## 3. Serialize and publish the bundle

Implement the writer in `src/libgit2/bundle.c` alongside the parser, keeping
public creation types separate from the parser's ownership model. Have the
internal writer emit through a write callback, as the parser reads through one,
so tests can exercise it against a `git_str` and a buffer or stream variant
stays cheap to add later. Only the file API is public in this PR.

1. Validate arguments, snapshot refs, and prepare the packbuilder.
2. Open the destination with `git_filebuf` and a regular file mode.
3. Write the version signature, any required object-format capability, selected
   OID/name entries, and the terminating blank line. Emit no prerequisites.
4. Stream packbuilder output with `git_packbuilder_foreach` into the same output.
5. Commit the filebuf only after header and pack writing succeed.

`git_filebuf` defines the replacement semantics: output goes to `<path>.lock`
and is renamed over the destination on commit, an existing lock fails with
`GIT_ELOCKED`, and cleanup removes only a lock created by this operation.
Document that rather than inventing a policy, and verify that validation errors,
write errors, and cancellation leave any pre-existing destination intact with
no lock created by this operation left behind. A pre-existing lock must remain
intact when returning `GIT_ELOCKED`.

Forward packbuilder progress through the options and propagate callback errors
according to libgit2 conventions. Ensure cleanup releases refs, walkers,
packbuilders, and output resources on every error path.

## 4. Add focused regression and interoperability coverage

Add a creation suite under `tests/libgit2/bundle/`, starting from the cases in
the earlier `create.c`, using generated output for round trips through the
existing parser, clone, and fetch paths.

Cover:

- One branch, multiple branches with shared history, and disconnected histories.
- Lightweight tags, annotated tags and tag chains, a tag-only selection; any
  supported tree/blob refs.
- Explicit symbolic or detached `HEAD`, omitted `HEAD`, duplicate selections,
  shorthand and invalid names, missing refs, unborn refs, and an empty
  selection.
- Header ordering follows caller order.
- SHA-1 and SHA-256, with appropriate build guards and format assertions.
- Complete object availability in a fresh destination, including commit parents
  and objects reachable only through tags.
- Rejection of shallow sources; failure on a missing required object (delete a
  loose object in a sandbox copy).
- Cancellation, output failures, overwrite of an existing destination, and no
  leftover lock created by this operation. An existing lock returns
  `GIT_ELOCKED` and its contents remain unchanged.

Keep Clar tests independent of a system Git installation. Check
interoperability once with a Git executable, via a small script next to
`tests/resources/bundle/generate.sh` or by hand, and report the results in the
PR rather than adding test infrastructure:

- `git bundle verify` and `git bundle list-heads` accept generated bundles.
- Git can clone and fetch generated bundles into fresh repositories.
- Advertised reference names and OIDs match equivalent Git-created bundles,
  regardless of header ordering. Test caller ordering separately in Clar.

Compare semantic contents rather than pack bytes.

## 5. Validate and prepare the PR

- Run the creation tests and existing bundle parser, transport-selection, clone,
  and fetch tests.
- Build with warnings treated as errors; run relevant sanitizer coverage on a
  working host and exercise SHA-256-enabled and default configurations.
- Run the broader packbuilder tests if its code changes. Use CI for platform
  coverage, particularly Windows file publication and callback behavior.
- Validate public API documentation and run `git diff --check`.
- Document the API with a minimal create-then-clone example and explicit limits.
- Describe actual interoperability and sanitizer results in the PR, and credit
  #7101 and #7221. Do not add a release changelog entry; the #7338 review
  states that maintainers generate the changelog during release preparation.
- `git rm plan.md` before opening the PR.

Completion means a documented public creation API, complete self-contained
output accepted by Git and libgit2, and tested failure/cancellation semantics.
Incremental creation should be a subsequent PR with its own prerequisite and
object-exclusion design.
