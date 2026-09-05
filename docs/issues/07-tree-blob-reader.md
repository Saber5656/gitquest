# Title

HEAD tree snapshot and blob content reader with file classification

## Summary

List the HEAD tree (`ls-tree`), classify entries (blob/symlink/submodule,
size), and read blob contents through a long-lived `cat-file --batch`
subprocess with strict size/encoding guards. Worktree files are never opened.

## Context

DESIGN §5.2 fixes the commands; §10.4 makes blob-only reads a security
property; §6.2 defines binary/size classification consumed by the detection
pipeline (19).

## Scope

`internal/gitio/tree.go` (+ tests):

```go
type TreeEntry struct {
    Path string; Mode uint32; SHA string; Size int64
    Kind EntryKind // Blob | Symlink | Submodule | Other
}
func ListTree(ctx context.Context, r *Runner, commitSHA string) ([]TreeEntry, []Warning, error)

type BlobReader struct{ /* wraps cat-file --batch stdin/stdout */ }
func NewBlobReader(ctx context.Context, r *Runner) (*BlobReader, error)
// Read returns decoded lines + classification for one blob. It takes the
// full TreeEntry (not a bare SHA) so it can reject non-Blob kinds itself —
// a symlink's target is stored as a blob object, so the SHA alone cannot
// distinguish it.
func (b *BlobReader) Read(entry TreeEntry, maxBytes int64) (BlobContent, error)
func (b *BlobReader) Close() error

type BlobContent struct {
    Size     int64
    Truncated bool      // size > maxBytes: Lines nil, only Size valid
    IsBinary bool       // NUL in first 8 KiB or >10% invalid UTF-8 in first 8 KiB
    Lines    []string   // nil when Truncated or IsBinary
    LongLines int       // count of lines truncated at 64 KiB
}
```

## Detailed Requirements

1. `ListTree`: `git ls-tree -r -z --long <commitSHA>`; parse
   `<mode> <type> <sha> <size>\t<path>\0`. Size for blobs is decimal; `-` for
   non-blobs → 0. Modes: `100644`/`100755` ⇒ Blob; `120000` ⇒ Symlink;
   `160000` ⇒ Submodule; else Other. Path validation identical to issue 06
   rule (relative, no `..`, valid UTF-8) — violations become Warnings, entry
   skipped.
2. Submodules and symlinks are returned by `ListTree` (callers need counts
   for scan warnings) but must never be content-read; `BlobReader.Read`
   rejects any `TreeEntry` with `Kind != Blob` with a typed error BEFORE
   writing anything to the `cat-file` process.
3. `BlobReader` protocol: write `<sha>\n`, read header
   `<sha> <type> <size>\n`, then exactly `<size>` bytes + `\n`. Enforce
   `maxBytes` BEFORE buffering: if header size > maxBytes, drain (io.CopyN to
   io.Discard) and return `Truncated=true` with `Size` set. Missing/`missing`
   header → typed error.
4. Line decoding: split on `\n` (accept `\r\n`, strip `\r`); strip UTF-8 BOM
   on first line; cap each line at 64 KiB (count in `LongLines`, truncate for
   analysis per DESIGN §10.5).
5. Binary heuristic exactly as DESIGN §6.2: NUL byte within first 8 KiB, or
   invalid-UTF-8 bytes > 10% of first 8 KiB.
6. One `BlobReader` per scan. Contract: safe for concurrent use, with calls
   serialized by an internal `sync.Mutex` (the batch protocol is inherently
   sequential). Document that throughput therefore comes from the pipeline
   (19) doing decode/detection in parallel AFTER Read returns, not from
   concurrent Reads.
7. Resource: internal copy buffer reused; no allocation proportional to blob
   size beyond the returned lines.

## Acceptance Criteria

- [ ] Fixture repo test: tree with regular file, executable, symlink,
      submodule (gitlink), nested UTF-8 path, file with tab in name (if git
      allows via `-z`) → classification table matches.
- [ ] Blob read: exact content round-trip; CRLF file; BOM file; binary file
      (contains NUL) flagged; 3 MiB file with maxBytes=2 MiB → Truncated,
      stream still usable for next request (drain correctness).
- [ ] Line > 64 KiB is truncated and counted.
- [ ] `Read` with a `TreeEntry` of Kind Symlink/Submodule → typed error, and
      nothing is written to the cat-file stdin (spy/pipe assertion).
- [ ] Concurrent Read calls from 2 goroutines succeed with no protocol
      corruption (internal mutex serializes; `-race` clean).

## Validation

```sh
go test -race ./internal/gitio/ -run 'TestListTree|TestBlobReader'
```

## Dependencies

- 04, 05.

## Non-goals

- Exclusion/candidate policy (18/19). Language detection (17).

## Design References

- DESIGN §5.2, §6.2, §10.4, §10.5.
