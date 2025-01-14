Plugins
=======

The plugins in this folder can be used to add additional capabilities to phan.
Add their relative path (.phan/plugins/...) to the `plugins` entry of .phan/config.php.

Plugin Documentation
--------------------

[Wiki Article: Writing Plugins For Phan](https://github.com/phan/phan/wiki/Writing-Plugins-for-Phan)

Plugin List
-----------

This section contains short descriptions of plugin files, and lists the issue types which they emit.

They are grouped into the following sections:

1. Plugins Affecting Phan Analysis
2. General-Use Plugins
3. Plugins Specific to Code Styles
4. Demo Plugins (Plugin authors should base new plugins off of these, if they don't see a similar plugin)

### 1. Plugins Affecting Phan Analysis

(More plugins will be added later, e.g. if they add new methods, add types to Phan's analysis of a return type, etc)

#### UnusedSuppressionPlugin.php

Warns if an `@suppress` annotation is no longer needed to suppress issue types on a function, method, closure, or class.
(Suppressions may stop being needed if Phan's analysis improves/changes in a release,
or if the relevant parts of the codebase fixed the bug/added annotations)
**This must be run with exactly one worker process**

- **UnusedSuppression**: `Element {FUNCTIONLIKE} suppresses issue {ISSUETYPE} but does not use it`
- **UnusedPluginSuppression**: `Plugin {STRING_LITERAL} suppresses issue {ISSUETYPE} on this line but this suppression is unused or suppressed elsewhere`
- **UnusedPluginFileSuppression**: `Plugin {STRING_LITERAL} suppresses issue {ISSUETYPE} in this file but this suppression is unused or suppressed elsewhere`

The following settings can be used in `.phan/config.php`:
 - `'plugin_config' => ['unused_suppression_ignore_list' => ['FlakyPluginIssueName']]` will make this plugin avoid emitting `Unused*Suppression` for a list of issue names.
 - `'plugin_config' => ['unused_suppression_whitelisted_only' => true]` will make this plugin report unused suppressions only for issues in `whitelist_issue_types`.

#### FFIAnalysisPlugin.php

This is only necessary if you are using [PHP 7.4's FFI (Foreign Function Interface) support](https://wiki.php.net/rfc/ffi)

This makes Phan infer that assignments to variables that originally contained CData will continue to be CData.

### 2. General-Use Plugins

These plugins are useful across a wide variety of code styles, and should give low false positives.
Also see [DollarDollarPlugin.php](#dollardollarpluginphp) for a meaningful real-world example.

#### AlwaysReturnPlugin.php

Checks if a function or method with a non-void return type will **unconditionally** return or throw.
This is stricter than Phan's default checks (Phan accepts a function or method that **may** return something, or functions that unconditionally throw).

#### DuplicateArrayKeyPlugin.php

Warns about common errors in php array keys and switch statements. Has the following checks (This is able to resolve global and class constants to their scalar values).

- **PhanPluginDuplicateArrayKey**: a duplicate or equivalent array key literal.

  (E.g `[2 => "value", "other" => "s", "2" => "value2"]` duplicates the key `2`)
- **PhanPluginDuplicateArrayKeyExpression**: `Duplicate/Equivalent dynamic array key expression ({CODE}) detected in array - the earlier entry will be ignored if the expression had the same value.`
  (E.g. `[$x => 'value', $y => "s", $y => "value2"]`)
- **PhanPluginDuplicateSwitchCase**: a duplicate or equivalent case statement.

  (E.g `switch ($x) { case 2: echo "A\n"; break; case 2: echo "B\n"; break;}` duplicates the key `2`. The later case statements are ignored.)
- **PhanPluginDuplicateSwitchCaseLooseEquality**: a case statement that is loosely equivalent to an earlier case statement.

  (E.g `switch ('foo') { case 0: echo "0\n"; break; case 'foo': echo "foo\n"; break;}` has `0 == 'foo'`, and echoes `0` because of that)
- **PhanPluginMixedKeyNoKey**: mixing array entries of the form [key => value,] with entries of the form [value,].

  (E.g. `['key' => 'value', 'othervalue']` is often found in code because the key for `'othervalue'` was forgotten)

#### PregRegexCheckerPlugin

This plugin checks for invalid regexes.
This plugin is able to resolve literals, global constants, and class constants as regexes.

- **PhanPluginInvalidPregRegex**: The provided regex is invalid, according to PHP.
- **PhanPluginInvalidPregRegexReplacement**: The replacement string template of `preg_replace` refers to a match group that doesn't exist. (e.g. `preg_replace('/x(a)/', 'y$2', $strVal)`)

#### PrintfCheckerPlugin

Checks for invalid format strings, incorrect argument counts, and unused arguments in printf calls.
Additionally, warns about incompatible union types (E.g. passing `string` for the argument corresponding to `%d`)
This plugin is able to resolve literals, global constants, and class constants as format strings.


- **PhanPluginPrintfNonexistentArgument**: `Format string {STRING_LITERAL} refers to nonexistent argument #{INDEX} in {STRING_LITERAL}`
- **PhanPluginPrintfNoArguments**: `No format string arguments are given for {STRING_LITERAL}, consider using {FUNCTION} instead`
- **PhanPluginPrintfNoSpecifiers**: `None of the formatting arguments passed alongside format string {STRING_LITERAL} are used`
- **PhanPluginPrintfUnusedArgument**: `Format string {STRING_LITERAL} does not use provided argument #{INDEX}`
- **PhanPluginPrintfNotPercent**: `Format string {STRING_LITERAL} contains something that is not a percent sign, it will be treated as a format string '{STRING_LITERAL}' with padding. Use %% for a literal percent sign, or '{STRING_LITERAL}' to be less ambiguous`
  (Usually a typo, e.g. `printf("%s is 20% done", $taskName)` treats `% d` as a second argument)
- **PhanPluginPrintfWidthNotPosition**: `Format string {STRING_LITERAL} is specifying a width({STRING_LITERAL}) instead of a position({STRING_LITERAL})`
- **PhanPluginPrintfIncompatibleSpecifier**: `Format string {STRING_LITERAL} refers to argument #{INDEX} in different ways: {DETAILS}` (e.g. `"%1$s of #%1$d"`. May be an off by one error.)
- **PhanPluginPrintfIncompatibleArgumentTypeWeak**: `Format string {STRING_LITERAL} refers to argument #{INDEX} as {DETAILS}, so type {TYPE} is expected. However, {FUNCTION} was passed the type {TYPE} (which is weaker than {TYPE})`
- **PhanPluginPrintfIncompatibleArgumentType**: `Format string {STRING_LITERAL} refers to argument #{INDEX} as {DETAILS}, so type {TYPE} is expected, but {FUNCTION} was passed incompatible type {TYPE}`
- **PhanPluginPrintfVariableFormatString**: `Code {CODE} has a dynamic format string that could not be inferred by Phan`

Note (for projects using `gettext`):
Subclassing this plugin (and overriding `gettextForAllLocales`) will allow you to analyze translations of a project for compatibility.
This will require extra work to set up.
See [PrintfCheckerPlugin's source](./PrintfCheckerPlugin.php) for details.

#### UnreachableCodePlugin.php

Checks for syntactically unreachable statements in the global scope or function bodies.
(E.g. function calls after unconditional `continue`/`break`/`throw`/`return`/`exit()` statements)

- **PhanPluginUnreachableCode**: `Unreachable statement detected`

#### Unused variable detection

This is now built into Phan itself, and can be enabled via `--unused-variable-detection`.

#### InvokePHPNativeSyntaxCheckPlugin.php

This invokes `php --no-php-ini --syntax-check $analyzed_file_path` for you. (See
This is useful for cases Phan doesn't cover (e.g. [Issue #449](https://github.com/phan/phan/issues/449) or [Issue #277](https://github.com/phan/phan/issues/277)).

Note: This may double the time Phan takes to analyze a project. This plugin can be safely used along with `--processes N`.

This does not run on files that are parsed but not analyzed.

Configuration settings can be added to `.phan/config.php`:

```php
    'plugin_config' => [
        // A list of 1 or more PHP binaries (Absolute path or program name found in $PATH)
        // to use to analyze your files with PHP's native `--syntax-check`.
        //
        // This can be used to simultaneously run PHP's syntax checks with multiple PHP versions.
        // e.g. `'plugin_config' => ['php_native_syntax_check_binaries' => ['php72', 'php70', 'php56']]`
        // if all of those programs can be found in $PATH

        // 'php_native_syntax_check_binaries' => [PHP_BINARY],

        // The maximum number of `php --syntax-check` processes to run at any point in time
        // (Minimum: 1. Default: 1).
        // This may be temporarily higher if php_native_syntax_check_binaries
        // has more elements than this process count.
        'php_native_syntax_check_max_processes' => 4,
    ],
```

If you wish to make sure that analyzed files would be accepted by those PHP versions
(Requires that php72, php70, and php56 be locatable with the `$PATH` environment variable)

#### UseReturnValuePlugin.php

This plugin warns when code fails to use the return value of internal functions/methods such as `sprintf` or `array_merge` or `Exception->getCode()`.
(functions/methods where the return value should almost always be used)

- **PhanPluginUseReturnValueInternalKnown**: `Expected to use the return value of the internal function/method {FUNCTION}`,

This plugin also has a dynamic mode(disabled by default and slow) where it will warn if a function or method's return value is unused.
This checks if the function/method's return value is used 98% or more of the time, then warns about the remaining places where the return value was unused.

- **PhanPluginUseReturnValue**: `Expected to use the return value of the user-defined function/method {FUNCTION} - {SCALAR}%% of calls use it in the rest of the codebase`,
- **PhanPluginUseReturnValueInternal**: `Expected to use the return value of the internal function/method {FUNCTION} - {SCALAR}%% of calls use it in the rest of the codebase`,

See [UseReturnValuePlugin.php](./UseReturnValuePlugin.php) for configuration options.

#### PHPUnitAssertionPlugin.php

This plugin will make Phan infer side effects from calls to some of the helper methods that PHPUnit provides in test cases.

- Infer that a condition is truthy from `assertTrue()` and `assertNotFalse()` (e.g. `assertTrue($x instanceof MyClass)`)
- Infer that a condition is null/not null from `assertNull()` and `assertNotNull()`
- Infer class types from `assertInstanceOf(MyClass::class, $actual)`
- Infer types from `assertInternalType($expected, $actual)`
- Infer that $actual has the exact type of $expected after calling `assertSame($expected, $actual)`
- Other methods aren't supported yet.

#### EmptyStatementListPlugin.php

This file checks for empty statement lists in loops/branches.
Due to Phan's AST rewriting for easier analysis, this may miss some edge cases for if/elseif.

- **PhanPluginEmptyStatementDoWhileLoop** `Empty statement list statement detected for the do-while loop`
- **PhanPluginEmptyStatementForLoop** `Empty statement list statement detected for the for loop`
- **PhanPluginEmptyStatementForeachLoop** `Empty statement list statement detected for the foreach loop`
- **PhanPluginEmptyStatementIf**: `Empty statement list statement detected for the last if/elseif statement`
- **PhanPluginEmptyStatementTryBody** `Empty statement list statement detected for the try statement's body`
- **PhanPluginEmptyStatementTryFinally** `Empty statement list statement detected for the try's finally body`
- **PhanPluginEmptyStatementWhileLoop** `Empty statement list statement detected for the while loop`

### 3. Plugins Specific to Code Styles

These plugins may be useful to enforce certain code styles,
but may cause false positives in large projects with different code styles.

#### NonBool

##### NonBoolBranchPlugin.php

- **PhanPluginNonBoolBranch** Warns if an expression which has types other than `bool` is used in an if/else if.

  (E.g. warns about `if ($x)`, where $x is an integer. Fix by checking `if ($x != 0)`, etc.)

##### NonBoolInLogicalArithPlugin.php

- **PhanPluginNonBoolInLogicalArith** Warns if an expression where the left/right-hand side has types other than `bool` is used in a binary operation.

  (E.g. warns about `if ($x && $x->fn())`, where $x is an object. Fix by checking `if (($x instanceof MyClass) && $x->fn())`)

#### HasPHPDocPlugin.php

Checks if an element (class or property) has a PHPDoc comment,
and that Phan can extract a plaintext summary/description from that comment.

- **PhanPluginNoCommentOnClass**: `Class {CLASS} has no doc comment`
- **PhanPluginDescriptionlessCommentOnClass**: `Class {CLASS} has no readable description: {STRING_LITERAL}`
- **PhanPluginNoCommentOnFunction**: `Function {FUNCTION} has no doc comment`
- **PhanPluginDescriptionlessCommentOnFunction**: `Function {FUNCTION} has no readable description: {STRING_LITERAL}`
- **PhanPluginNoCommentOnPublicProperty**: `Public property {PROPERTY} has no doc comment` (Also exists for Private and Protected)
- **PhanPluginDescriptionlessCommentOnPublicProperty**: `Public property {PROPERTY} has no readable description: {STRING_LITERAL}` (Also exists for Private and Protected)

Warnings about method verbosity also exist, many categories may need to be completely disabled due to the large number of method declarations in a typical codebase:

- Warnings are not emitted for `@internal` methods.
- Warnings are not emitted for methods that override methods in the parent class.
- Warnings can be suppressed based on the method FQSEN with `plugin_config => [..., 'has_phpdoc_method_ignore_regex' => (a PCRE regex)]`

  (e.g. to suppress issues about tests, or about missing documentation about getters and setters, etc.)

The warning types for methods are below:

- **PhanPluginNoCommentOnPublicMethod**: `Public method {METHOD} has no doc comment` (Also exists for Private and Protected)
- **PhanPluginDescriptionlessCommentOnPublicMethod**: `Public method {METHOD} has no readable description: {STRING_LITERAL}` (Also exists for Private and Protected)

#### InvalidVariableIssetPlugin.php

Warns about invalid uses of `isset`. This README documentation may be inaccurate for this plugin.

- **PhanPluginInvalidVariableIsset** : Forces all uses of `isset` to be on arrays or variables.

  E.g. it will warn about `isset(foo()['key'])`, because foo() is not a variable or an array access.
- **PhanUndeclaredVariable**: Warns if `$array` is undeclared in `isset($array[$key])`

#### NoAssertPlugin.php

Discourages the usage of assert() in the analyzed project.
See https://secure.php.net/assert

- **PhanPluginNoAssert**: `assert() is discouraged. Although phan supports using assert() for type annotations, PHP's documentation recommends assertions only for debugging, and assert() has surprising behaviors.`

#### NotFullyQualifiedUsagePlugin.php

Encourages the usage of fully qualified global functions and constants (slightly faster, especially for functions such as `strlen`, `count`, etc.)

- **PhanPluginNotFullyQualifiedFunctionCall**: `Expected function call to {FUNCTION}() to be fully qualified or have a use statement but none were found in namespace {NAMESPACE}`
- **PhanPluginNotFullyQualifiedOptimizableFunctionCall**: `Expected function call to {FUNCTION}() to be fully qualified or have a use statement but none were found in namespace {NAMESPACE} (opcache can optimize fully qualified calls to this function in recent php versions)`
- **PhanPluginNotFullyQualifiedGlobalConstant**: `Expected usage of {CONST} to be fully qualified or have a use statement but none were found in namespace {NAMESPACE}`

#### NumericalComparisonPlugin.php

Enforces that loose equality is used for numeric operands (e.g. `2 == 2.0`), and that strict equality is used for non-numeric operands (e.g. `"2" === "2e0"` is false).

- **PhanPluginNumericalComparison**: nonnumerical values compared by the operators '==' or '!=='; numerical values compared by the operators '===' or '!=='

#### PHPUnitNotDeadCodePlugin.php

Marks unit tests and dataProviders of subclasses of PHPUnit\Framework\TestCase as referenced.
Avoids false positives when `--dead-code-detection` is enabled.

(Does not emit any issue types)

#### SleepCheckerPlugin.php

Warn about returning non-arrays in [`__sleep`](https://secure.php.net/__sleep),
as well as about returning array values with invalid property names in `__sleep`.

- **SleepCheckerInvalidReturnStatement`**: `__sleep must return an array of strings. This is definitely not an array.`
- **SleepCheckerInvalidReturnType**: `__sleep is returning {TYPE}, expected string[]`
- **SleepCheckerInvalidPropNameType**: `__sleep is returning an array with a value of type {TYPE}, expected string`
- **SleepCheckerInvalidPropName**: `__sleep is returning an array that includes {PROPERTY}, which cannot be found`
- **SleepCheckerMagicPropName**: `__sleep is returning an array that includes {PROPERTY}, which is a magic property`
- **SleepCheckerDynamicPropName**: `__sleep is returning an array that includes {PROPERTY}, which is a dynamically added property (but not a declared property)`
- **SleepCheckerPropertyMissingTransient**: `Property {PROPERTY} that is not serialized by __sleep should be annotated with @transient or @phan-transient`,

#### UnknownElementTypePlugin.php

Warns about elements containing unknown types (function/method/closure return types, parameter types)

- **PhanPluginUnknownMethodReturnType**: `Method {METHOD} has no declared or inferred return type`
- **PhanPluginUnknownMethodParamType**: `Method {METHOD} has no declared or inferred parameter type for ${PARAMETER}`
- **PhanPluginUnknownFunctionReturnType**: `Function {FUNCTION} has no declared or inferred return type`
- **PhanPluginUnknownFunctionParamType**: `Function {FUNCTION} has no declared or inferred return type for ${PARAMETER}`
- **PhanPluginUnknownClosureReturnType**: `Closure {FUNCTION} has no declared or inferred return type`
- **PhanPluginUnknownClosureParamType**: `Closure {FUNCTION} has no declared or inferred return type for ${PARAMETER}`
- **PhanPluginUnknownPropertyType**: `Property {PROPERTY} has an initial type that cannot be inferred`

#### DuplicateExpressionPlugin.php

This plugin checks for duplicate expressions in a statement
that are likely to be a bug. (e.g. `expr1 == expr`)

- **PhanPluginDuplicateExpressionAssignment**: `Both sides of the assignment {OPERATOR} are the same: {CODE}`
- **PhanPluginDuplicateExpressionBinaryOp**: `Both sides of the binary operator {OPERATOR} are the same: {CODE}`
- **PhanPluginDuplicateConditionalTernaryDuplication**: `"X ? X : Y" can usually be simplified to "X ?: Y". The duplicated expression X was {CODE}`
- **PhanPluginDuplicateConditionalNullCoalescing**: `"isset(X) ? X : Y" can usually be simplified to "X ?? Y" in PHP 7. The duplicated expression X was {CODE}`
- **PhanPluginBothLiteralsBinaryOp**: `Suspicious usage of a binary operator where both operands are literals. Expression: {CODE} {OPERATOR} {CODE} (result is {CODE})` (e.g. warns about `null == 'a literal` in `$x ?? null == 'a literal'`)
- **PhanPluginDuplicateConditionalUnnecessary**: `"X ? Y : Y" results in the same expression Y no matter what X evaluates to. Y was {CODE}`

#### WhitespacePlugin.php

This plugin checks for unexpected whitespace in PHP files.

- **PhanPluginWhitespaceCarriageReturn**: `The first occurrence of a carriage return ("\r") was seen here. Running "dos2unix" can fix that.`
- **PhanPluginWhitespaceTab**: `The first occurrence of a tab was seen here. Running "expand" can fix that.`
- **PhanPluginWhitespaceTrailing**: `The first occurrence of trailing whitespace was seen here.`

#### InlineHTMLPlugin.php

This plugin checks for unexpected inline HTML.

This can be limited to a subset of files with an `inline_html_whitelist_regex` - e.g. `@^(src/|lib/)@`.

Files can be excluded with `inline_html_blacklist_regex`, e.g. `@(^src/templates/)|(\.html$)@`

- **PhanPluginInlineHTML**: `Saw inline HTML between the first and last token: {STRING_LITERAL}`
- **PhanPluginInlineHTMLLeading**: `Saw inline HTML at the start of the file: {STRING_LITERAL}`
- **PhanPluginInlineHTMLTrailing**: `Saw inline HTML at the end of the file: {STRING_LITERAL}`

#### SuspiciousParamOrderPlugin.php

This plugin guesses if arguments to a function call are out of order, based on heuristics on the name in the expression (e.g. variable name).
This will only warn if the argument types are compatible with the alternate parameters being suggested.
This may be useful when analyzing methods with long parameter lists.

E.g. warns about invoking `function example($first, $second, $third)` as `example($mySecond, $myThird, $myFirst)`

- **PhanPluginSuspiciousParamOrder**: `Suspicious order for arguments named {DETAILS} - These are being passed to parameters {DETAILS} of {FUNCTION} defined at {FILE}:{LINE}`
- **PhanPluginSuspiciousParamOrderInternal**: `Suspicious order for arguments named {DETAILS} - These are being passed to parameters {DETAILS}`

#### PossiblyStaticMethodPlugin.php

Checks if a method can be made static without causing any errors.

- **PhanPluginPossiblyStaticPublicMethod**: `Public method {PROPERTY} can be static` (Also exists for Private and Protected)

Warnings may need to be completely disabled due to the large number of method declarations in a typical codebase:

- Warnings are not emitted for methods that override methods in the parent class.
- Warnings are not emitted for methods that are overridden in child classes.
- Warnings can be suppressed based on the method FQSEN with `plugin_config => [..., 'possibly_static_method_ignore_regex' => (a PCRE regex)]`

#### PHPDocToRealTypesPlugin.php

This plugin suggests real types that can be used instead of phpdoc types.
Currently, this just checks param and return types.
Some of the suggestions made by this plugin will cause inheritance errors.

This doesn't suggest changes if classes have subclasses (but this check doesn't work when inheritance involves traits).
`PHPDOC_TO_REAL_TYPES_IGNORE_INHERITANCE=1` can be used to force this to check **all** methods and emit issues.

This also supports `--automatic-fix` to add the types to the real type signatures.

- **PhanPluginCanUseReturnType**: `Can use {TYPE} as a return type of {METHOD}`
- **PhanPluginCanUseNullableReturnType**: `Can use {TYPE} as a return type of {METHOD}` (useful if there is a minimum php version of 7.1)
- **PhanPluginCanUsePHP71Void**: `Can use php 7.1's void as a return type of {METHOD}` (useful if there is a minimum php version of 7.1)

This supports `--automatic-fix`.
- `PHPDocRedundantPlugin` will be useful for cleaning up redundant phpdoc after real types were added.
- `PreferNamespaceUsePlugin` can be used to convert types from fully qualified types back to unqualified types ()

#### PHPDocRedundantPlugin.php

This plugin warns about function/method/closure phpdoc that does nothing but repeat the information in the type signature.
E.g. this will warn about `/** @return void */ function () : void {}` and `/** */`, but not `/** @return void description of what it does or other annotations */`

This supports `--automatic-fix`

- **PhanPluginRedundantFunctionComment**: `Redundant doc comment on function {FUNCTION}(): {COMMENT}`
- **PhanPluginRedundantMethodComment**: `Redundant doc comment on method {METHOD}(): {COMMENT}`
- **PhanPluginRedundantClosureComment**: `Redundant doc comment on closure {FUNCTION}: {COMMENT}`
- **PhanPluginRedundantReturnComment**: `Redundant @return {TYPE} on function {FUNCTION}: {COMMENT}`

#### PreferNamespaceUsePlugin.php

This plugin suggests using `ClassName` instead of `\My\Ns\ClassName` when there is a `use My\Ns\ClassName` annotation (or for uses in namespace `\My\Ns`)
Currently, this only checks **real** (not phpdoc) param/return annotations.

- **PhanPluginPreferNamespaceUseParamType**: `Could write param type of ${PARAMETER} of {FUNCTION} as {TYPE} instead of {TYPE}`
- **PhanPluginPreferNamespaceUseReturnType**: `Could write return type of {FUNCTION} as {TYPE} instead of {TYPE}`

##### StrictComparisonPlugin.php

This plugin warns about non-strict comparisons. It warns about the following issue types:

1. Using `in_array` and `array_search` without explicitly passing true or false to `$strict`.
2. Using equality or comparison operators when both sides are possible objects.

- **PhanPluginComparisonNotStrictInCall**: `Expected {FUNCTION} to be called with a third argument for {PARAMETER} (either true or false)`
- **PhanPluginComparisonObjectEqualityNotStrict**: `Saw a weak equality check on possible object types {TYPE} and {TYPE} in {CODE}`
- **PhanPluginComparisonObjectOrdering**: `Saw a weak equality check on possible object types {TYPE} and {TYPE} in {CODE}`

### 4. Demo plugins:

These files demonstrate plugins for Phan.

#### DemoPlugin.php

Look at this class's documentation if you want an example to base your plugin off of.
Generates the following issue types under the types:

- **DemoPluginClassName**: a declared class isn't called 'Class'
- **DemoPluginFunctionName**: a declared function isn't called `function`
- **DemoPluginMethodName**: a declared method isn't called `function`
  PHP's default checks(`php -l` would catch the class/function name types.)
- **DemoPluginInstanceof**: codebase contains `(expr) instanceof object` (usually invalid, and `is_object()` should be used instead. That would actually be a check for `class object`).

#### DollarDollarPlugin.php

Checks for complex variable access expressions `$$x`, which may be hard to read, and make the variable accesses hard/impossible to analyze.

- **PhanPluginDollarDollar**: Warns about the use of $$x, ${(expr)}, etc.

### 5. Third party plugins

- https://github.com/Drenso/PhanExtensions is a third party project with several plugins to do the following:

  - Analyze Symfony doc comment annotations.
  - Mark elements in inline doc comments (which Phan doesn't parse) as referencing types from `use statements` as not dead code.

- https://github.com/TysonAndre/PhanTypoCheck checks all tokens of PHP files for typos, including within string literals.
  It is also able to analyze calls to `gettext()`.

### 6. Self-analysis plugins:

#### PhanSelfCheckPlugin.php

This plugin checks for invalid calls to `PluginV2::emitIssue`, `Issue::maybeEmit()`, etc.
This is useful for developing Phan and Phan plugins.

- **PhanPluginTooFewArgumentsForIssue**: `Too few arguments for issue {STRING_LITERAL}: expected {COUNT}, got {COUNT}`
- **PhanPluginTooManyArgumentsForIssue**: `Too many arguments for issue {STRING_LITERAL}: expected {COUNT}, got {COUNT}`
- **PhanPluginUnknownIssueType**: `Unknown issue type {STRING_LITERAL} in a call to {METHOD}(). (may be a false positive - check if the version of Phan running PhanSelfCheckPlugin is the same version that the analyzed codebase is using)`
