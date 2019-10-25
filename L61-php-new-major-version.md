gRPC PHP 2.x Release 
----
* Author(s): stanleycheung
* Approver: TBD
* Status: Draft
* Implemented in: PHP
* Last updated: 2019-10-25
* Discussion at: 


## Abstract

gRPC PHP implementation will do a breaking change in a TBD future release,
which also forces a major version increase because we follow semantic
versioning. This document summarizes what are the changes being made and the
reasons why they are needed. It also describes the impact on the users and
documents the migration steps.


## Background

[PHP 5.x](https://www.php.net/supported-versions.php) has been in the
end-of-life state since Jan 2019 and will no longer receive security fixes.
PHP 7 (currently PHP 7.2) is the major active version that PHP supports. We are
proposing that we drop support to PHP 5 and only support PHP 7 going forward.
We also propose to take this opportunity and evaluate whether any API
breaking changes will also be introduced at this time.


## Proposal

We propose that we pick a future point release X of the regular gRPC C core
release cycle to execute a major version bump for the PHP extension that,
after that point, the 2.X.0 releases, and any all point releases afterwards,
will only support PHP 7 and above. The 1.X line will continue to support PHP 5
for a period of time (length of time TBD).


#### Change 1: Remove PHP 5 support

[PHP 5.x](https://www.php.net/supported-versions.php) has been in the
end-of-life state since Jan 2019 and will no longer receive security fixes.
PHP 7 (currently PHP 7.2) is the major active version that PHP supports. We are
proposing that we drop support to PHP 5 and only support PHP 7 going forward.

This will allow us to better spend our resources and maintaining the PHP
extension and not having to worry about PHP 5 compatibility.

By [some measures](https://blog.packagist.com/php-versions-stats-2019-1-edition/),
PHP 5.x usage has just dipped below 10%, plus the fact that PHP 5.x will not be
receiving security fixes, we felt that this is about time we drop PHP 5 support.
This will also allow us to finally be able to take advantage of some new PHP 7
syntax like
[return type declarations](https://www.php.net/manual/en/migration70.new-features.php#migration70.new-features.return-type-declarations)
and to make our PHP codebase more modern.




#### Change 2: Potential Breaking API changes

Since to achieve Change 1 above, we need to execute a major version bump to
adhere to semantic versioning, we might as well take this chance to review our
API surface to see if there is any need to introduce some breaking API changes
also at this time to minimize future disruption.

Potential candidates:
 - [Interceptor API](L31-php-intercetor-api-change.md)
 - More TBD




## Rationale

#### Change 1

 - [PHP 5.x](https://www.php.net/supported-versions.php) has reached the
 end-of-life state since Jan 2019 and will not be receiving security fixes.
 - PHP 5.x overall usage, by several measures, e.g. Composer usage, Google
 Cloud usage by traffic, or by projects have dipped below certain level that
 we feel that it's appropriate to require PHP 7 going forward.
 - Resources can be better invested into improving and maintaining the PECL
 extension if we no longer have to worry about PHP 5 compatibility.
 - We can start introducing new syntax from PHP 7 to modernize our library.


#### Change 2

TBD


## Implementation

We will be removing PHP 5 support from our `grpc` PECL extension from the
future `2.x` mainline.


## Migration instructions for users

To upgrade from 1.x to 2.x of the PECL extension, user will just upgrade via
`pecl install` normally.

For the `grpc/grpc` composer package, user will need to update their
`composer.json` file to

```
  "require": {
    "grpc/grpc": "^v2.0.0"
  },
```

The `1.x` line of both the PECL extension and the Packagist package will still
be maintained for PHP 5.x users for a TBD amount of time.
