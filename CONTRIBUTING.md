# Contributing

Discuss a focused change in the [issue tracker](https://github.com/altophp/code-diff/issues)
and include regression tests with pull requests. The library supports PHP 8.3;
the current PHPUnit development dependency requires PHP 8.4 or later.

Run `composer install`, then `vendor/bin/php-cs-fixer fix --dry-run --diff --sequential`,
`composer sa`, and `composer test`. `composer cs` applies fixes, so review its diff
before committing. Documentation examples should preserve exact patch newlines,
show expected output, and keep filesystem access in the caller.
