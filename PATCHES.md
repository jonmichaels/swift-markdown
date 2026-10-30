# Patches in this fork

This is [swiftlang/swift-markdown](https://github.com/swiftlang/swift-markdown) with fixes that
[live-markdown](https://github.com/jonmichaels/live-markdown-editor) needs before they are in an upstream release. Each
patch is one commit on the branch `lme-0.9`, and each release of this fork is a tag `<upstream version>-lme.<n>`. When
upstream releases a fix, the patch is dropped and live-markdown goes back to upstream.

| Tag | Upstream base | Patches |
|---|---|---|
| `0.9.0-lme.1` | `0.9.0` (`25cb61d3482054b09ae76ca4f281b1bfe7fe5a43`) | 1 |

## 1. Parsing does not scale across threads (`CommonMarkConverter.swift`)

**Symptom.** Parsing many documents at once barely uses more than one core. In live-markdown's test (15,000 synthetic
notes, 52.7 MB, `MarkdownParser.parse` and metadata, release build, Apple silicon with 14 cores),
`DispatchQueue.concurrentPerform` gave only 1.3x: 7.4 s on one core, 5.9 s on 14.

**Cause.** `MarkupConverterState.nodeType` maps each cmark node to `CommonMarkNodeType` with
`CommonMarkNodeType(rawValue: String(cString: cmark_node_get_type_string(node)))`. A `String`-raw-value enum's
`init(rawValue:)` compiles to a switch over string cases, `_findStringSwitchCaseWithCache`, which retains and
releases one global cache on every call. The converter asks for the node type several times per node, so every
thread hits the same atomic reference count. A `sample` of the parallel run found two thirds of all samples in
`swift_retain` / `swift_release` called from `_findStringSwitchCaseWithCache` under `CommonMarkNodeType.init(rawValue:)`.
On one thread it is about 3 % of the time.

**Fix.** `CommonMarkNodeType.init?(cString:)` compares cmark's type string with `strcmp` against each case's raw value
(static string literals, so nothing is allocated or retained), most frequent types first. `nodeType` calls it instead
of `init(rawValue:)`. The enum and its raw values are unchanged; an unknown type still traps with the same message.
cmark's numeric node type is not used, because upstream's comment notes it is not reliable for extension nodes.

**Result.** Same test: 14 cores 5.9 s → 0.69 s (8.6x faster, 9.6x over one core); one core 7.4 s → 6.6 s.
This fork's 300 tests pass, and so do live-markdown's 1,416.

**Upstream.** Not yet reported (2026-10-30).
