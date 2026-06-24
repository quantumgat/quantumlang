# qtlc1 Core Text Design

Status: active qtlc1 compiler contract

## Boundary

Text literals, interpolation, formatting, and common text operations are
language/core capabilities. They require no import.

Codec and security policies remain extension surfaces:

```text
codec: jsonEscape, jsonUnescape, urlEncode, urlDecode, htmlEscape, shellEscape
security: constantTimeEq, constantTimeContains, redact, auditEscape
```

They use receiver syntax after their capability module is available, but they
are not silently treated as universal primitive behavior.

## Literal Surface

```qn
let text = "hello"
let bytesValue = b"QCAP"
let message = "user: {user.name}"
let braces = "{{value}}"
```

Required escapes:

```text
\\ \" \' \n \r \t \0 \b \f \v \xNN \uNNNN \u{...} {{ }}
```

Required later literal forms:

```text
r"raw text"
"""multiline text"""
r"""raw multiline text"""
```

`TextContract.remainingLiteralWork` remains true until raw, multiline, full
escape decoding, and static format-plan lowering are executable.

## Canonical Import-Free Methods

```text
measure:
  byteLength length charLength graphemeLength

validate:
  isEmpty isAscii isUtf8

compare/search:
  eq compare startsWith endsWith contains
  startsWithIgnoreCase endsWithIgnoreCase containsIgnoreCase
  find indexOf lastIndexOf findIndex findLastIndex
  findByte findChar findGrapheme findIgnoreCase findInRange count

slice/view:
  slice sliceBytes sliceChars sliceGraphemes substring
  prefix suffix take takeLast drop dropLast
  trim trimStart trimEnd splitOnce rsplitOnce

transform:
  concat repeat replace lower upper asciiLower asciiUpper
  capitalize title caseFold normalize

convert/unicode:
  asBytes toBytes toString displayWidth unicodeCategory
```

Compatibility aliases:

```text
len -> length
lenBytes -> byteLength
lenChars -> charLength
graphemeLen -> graphemeLength
```

New qtlc1 source should use canonical names.

## Return Contract

```text
indexOf / lastIndexOf -> i64
  legacy sentinel API; -1 means missing

findIndex / findLastIndex / findByte / findChar
findGrapheme / findIgnoreCase / findInRange -> Option<usize>

splitOnce / rsplitOnce -> Option<TextSplit>
asBytes -> BytesView
toBytes -> bytes
```

## Native Contract

- Literal length/search folds during compilation when operands are constant.
- `length` is a direct carrier length load.
- `isEmpty` is a length comparison.
- Slice, trim, prefix, suffix, take, and drop return views.
- No reflection, dynamic dispatch, hidden map, or runtime method lookup.
- Constant format strings become static format plans.
- Allocating transforms allocate only when required.
- `replace` uses one output allocation, returns the original view when no
  match exists, and is registered in the live runtime ABI. Historical frozen
  ABI baselines remain unchanged.

## Implementation Commands

```text
POST-QTLC1-CORE-TEXT-CATALOG-001
  status: implemented
  add grouped complete qtlc1 text method catalog
  add canonical names and compatibility aliases
  distinguish import-free core from codec/security extensions
  expose nativeReady, zeroCopy, and allocation facts

POST-QTLC1-CORE-TEXT-LITERAL-FORMAT-002
  status: implemented
  strict text and bytes escape validation with QLLEX0004
  raw r"..." and raw multiline r"""...""" literals
  multiline """...""" and f"""...""" interpolation
  compile constant templates and format specs into TextFormatPlan
  native literals use static read-only data with no runtime escape parsing

POST-QTLC0-TEXT-CANONICAL-ALIASES-003
  status: implemented
  add byteLength, charLength, graphemeLength and substring native aliases
  route aliases through the central qtlc0 text registry

POST-QTLC0-TEXT-REPLACE-NATIVE-004
  status: implemented
  implement QnTextReplace runtime kernel and live ABI registration
  add typecheck, lowering, native emit, executable, and assembly proofs
```
