# Patches

Alto Code Diff understands standard unified diffs and common Git headers. It operates on strings and associative arrays; your application remains responsible for file-system access.

## Emit a patch

`UnifiedEmitter::emit()` accepts a `DiffResult` or a multi-file `DiffBundle`.

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Alto\Code\Diff\Diff;
use Alto\Code\Diff\Patch\UnifiedEmitter;

$result = Diff::build()->compare(
    "Line one\nLine two\n",
    "Line one\nLine two changed\n",
);

$patch = (new UnifiedEmitter())->emit($result);
echo $patch;
```

A bare result uses `a` and `b` as labels. To control paths or represent multiple files, construct `DiffFile` objects and place them in a `DiffBundle`.

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Alto\Code\Diff\Diff;
use Alto\Code\Diff\Model\DiffBundle;
use Alto\Code\Diff\Model\DiffFile;
use Alto\Code\Diff\Patch\UnifiedEmitter;

$result = Diff::build()->compare("old\n", "new\n");
$file = new DiffFile(
    oldPath: 'src/example.txt',
    newPath: 'src/example.txt',
    result: $result,
    headers: [
        'diff' => 'diff --git a/src/example.txt b/src/example.txt',
        'index' => 'index 3367afd..3e75765 100644',
    ],
);

echo (new UnifiedEmitter())->emit(new DiffBundle([$file]));
```

Supported metadata keys are `diff`, `index`, `old_mode`, `new_mode`, `new_file_mode`, `deleted_file_mode`, `similarity_index`, `rename_from`, `rename_to`, `copy_from`, and `copy_to`.

## Parse a patch

`UnifiedParser::parse(string $patch): DiffBundle` validates hunk lengths and returns files, paths, headers, results, hunks, and edits. Leading `a/` and `b/` path prefixes are removed.

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Alto\Code\Diff\Patch\UnifiedParser;

$patch = <<<'PATCH'
diff --git a/example.txt b/example.txt
index 3367afd..3e75765 100644
--- a/example.txt
+++ b/example.txt
@@ -1 +1 @@
-old
+new
PATCH;

$bundle = (new UnifiedParser())->parse($patch);
$file = $bundle->files()[0];

printf("%s -> %s\n", $file->oldPath, $file->newPath);
```

The parser recognizes file modes, creation, deletion, rename, copy, similarity, and index headers. It preserves no-trailing-newline markers. It throws `ParseException` for malformed hunks and `BinaryInputException` for binary patch markers.

## Apply a single-file patch

`PatchApplier::apply(string $original, string $unifiedPatch): string` accepts exactly one patched file. An empty patch returns the original string.

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Alto\Code\Diff\Patch\PatchApplier;

$original = "Line one\nLine two\n";
$patch = <<<'PATCH'
--- a/example.txt
+++ b/example.txt
@@ -1,2 +1,2 @@
 Line one
-Line two
+Line two changed
PATCH;

$updated = (new PatchApplier())->apply($original, $patch);
echo $updated;
```

The constructor accepts `fuzz` and `maxBytes`, both defaulting to `0` and `5_000_000`. Fuzz searches that many lines before and after a hunk's expected position. `PatchApplyException` exposes the failed zero-based `hunkIndex`; `SizeLimitException` reports oversized source content.

## Apply a bundle

Use `applyBundle(array $files, DiffBundle $bundle): array` for multiple files. The input and result use `path => content` maps.

The method handles modifications, renames, creations from `/dev/null`, and deletions to `/dev/null`. It throws `PatchApplyException` when a required source path is missing or a hunk cannot be matched. The library returns updated content but never writes it to disk.


## Verify a complete round trip

This example keeps every file in memory. The package does not open the paths:

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Alto\Code\Diff\Diff;
use Alto\Code\Diff\Model\DiffBundle;
use Alto\Code\Diff\Model\DiffFile;
use Alto\Code\Diff\Patch\PatchApplier;
use Alto\Code\Diff\Patch\UnifiedEmitter;
use Alto\Code\Diff\Patch\UnifiedParser;

$files = ['a.txt' => "old\n", 'b.txt' => "keep\n"];
$change = new DiffFile('a.txt', 'a.txt', Diff::build()->compare($files['a.txt'], "new\n"));
$patch = (new UnifiedEmitter())->emit(new DiffBundle([$change]));
$bundle = (new UnifiedParser())->parse($patch);
$updated = (new PatchApplier())->applyBundle($files, $bundle);
echo json_encode($updated, JSON_THROW_ON_ERROR), "\n";
```

Output:

```text
{"a.txt":"new\n","b.txt":"keep\n"}
```

## Recover from a rejected patch

| Failure | What to check before retrying |
| --- | --- |
| `ParseException` | Preserve the patch headers, prefixes, and hunk counts; obtain a complete unified patch rather than guessing missing lines. |
| `BinaryInputException` | Supply a text patch; binary Git patches are not supported. |
| `PatchApplyException` | Match the original source revision and required path keys; inspect `hunkIndex` for a failed hunk. |
| `SizeLimitException` | Check input sizes and the configured limit before deliberately raising it. |

Fuzz searches nearby line positions; it is not conflict resolution and does not
ignore different source text. If the patch applies to another revision, regenerate
it against the intended base. Keep the returned map separate until your application
has decided how to persist it; the package performs no file writes.
