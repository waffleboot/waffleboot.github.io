---
title: "Errors in Go 1.13"
date: 2021-08-30T02:38:04+03:00
draft: false
---

# Errors in Go 1.13

## Whether to Wrap

**When adding additional context to an error,** either with `fmt.Errorf` or by implementing a custom type, you need to decide whether the new error should wrap the original. There is no single answer to this question; it depends on the context in which the new error is created. Wrap an error **to expose** it to callers. Do not wrap an error when doing so would expose **implementation** details.

As one example, imagine a `Parse` function which reads a complex data structure from an `io.Reader`. If an error occurs, we wish to report the line and column number at which it occurred. If the error occurs while reading from the `io.Reader`, we will want to wrap that error to allow inspection of the underlying problem. Since the caller **provided** the `io.Reader` to the function, it makes sense to expose the error produced by it.

In contrast, a function which makes several calls **to a database** probably should not return an error which unwraps to the result of one of those calls. If the database used by the function **is an implementation detail**, then exposing these errors **is a violation of abstraction**. For example, if the `LookupUser` function of your package `pkg` uses Go’s `database/sql` package, then it may encounter a `sql.ErrNoRows` error.

* If you return that error with   `fmt.Errorf("accessing DB: %v", err)` then a caller cannot look inside to find the `sql.ErrNoRows`.
* But if the function instead returns `fmt.Errorf("accessing DB: %w", err)`, then a caller could reasonably write

 ```
err := pkg.LookupUser(...)
if errors.Is(err, sql.ErrNoRows) …
```

At that point, the function **must** always return `sql.ErrNoRows` if you don’t want **to break** your clients, even if you **switch** to a different database package.

**In other words, wrapping an error makes that error part of your API.  
If you don’t want to commit to supporting that error as part of your API in the future, you shouldn’t wrap the error.**

## Customizing error tests with Is and As methods

The `errors.Is` function examines each error in a chain for a match with a target value. By default, an error matches the target if the two are equal. In addition, an error in the chain may declare that it matches a target by implementing an `Is` *method*.

It looks like about custom errors matching.

## Errors and package APIs

If we wish a function to return an **identifiable** error condition, such as “item not found,” we might return an error wrapping a **sentinel**.

```
var ErrNotFound = errors.New("not found")

// FetchItem returns an error wrapping ErrNotFound.
func FetchItem(name string) (*Item, error) {
  return nil, fmt.Errorf("%q: %w", name, ErrNotFound)
}
```

There are other existing patterns for providing errors which can be semantically examined by the caller, such as directly returning a sentinel value, a specific type, or a value which can be examined with a predicate function.

In all cases, care should be taken not to expose internal details to the user. As we touched on in “Whether to Wrap” above, when you return an error from another package you should convert the error to a form that does not expose the underlying error, unless you are willing to commit to returning that specific error in the future.

```
f, err := os.Open(filename)
if err != nil {
    // The *os.PathError returned by os.Open is an internal detail.
    // To avoid exposing it to the caller, repackage it as a new
    // error with the same text. We use the %v formatting verb, since
    // %w would permit the caller to unwrap the original *os.PathError.
    return fmt.Errorf("%v", err)
}
```

If a function is defined as returning an error **wrapping** some sentinel or type, do not return the underlying error **directly**.

```
var ErrPermission = errors.New("permission denied")

// returns an error wrapping ErrPermission
func DoSomething() error {
    // If we return ErrPermission directly, callers might come
    // to depend on the exact error value, writing code like this:
    //
    //     if err := pkg.DoSomething(); err == pkg.ErrPermission { … }
    //
    // This will cause problems if we want to add additional
    // context to the error in the future. To avoid this, we
    // return an error wrapping the sentinel so that users must
    // always unwrap it:
    //
    //     if err := pkg.DoSomething(); errors.Is(err, pkg.ErrPermission) { ... }
    return fmt.Errorf("%w", ErrPermission)
}
```
