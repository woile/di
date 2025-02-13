# Scopes

Scopes are one of the fundamental concepts in dependency injection.
Some dependency injection frameworks provide fixed scopes, for example:

- Singleton: only one instance is created
- Request: in web frameworks, this could be the lifetime of a request
- Prototype: re-initialized every time it is needed

`di` generalizes this concept by putting control of scopes into the hands of the users / implementers: a scope in `di` is identified by any hashable value (a string, enum, int, etc.) and entering / exiting scopes is handled via context managers:

```python
async with container.enter_scope("app"):
    async with container.enter_scope("request"):
        async with container.enter_scope("foo, bar, baz!"):
```

Scopes provide a framework for several other important features:

- Dependency lifespans
- Dependency value sharing

Every dependency is linked to a scope.
When a scope exits, all dependencies linked to it are destroyed (if they have teardown, the teardown is run) and their value is removed from the cache.
This means that dependencies scoped to an outer scope cannot depend on dependencies scoped to an inner scope:

```Python
--8<-- "docs_src/invalid_scope_dependance.py"
```

This example will fail with `di.exceptions.ScopeViolationError` because an `DBConnection` is scoped to `"app"` so it cannot depend on `Request` which is scoped to `"request"`.

The order of the scopes is determined by the `scopes` parameter to `Container.solve`.
If you've used Pytest fixtures before, you're already familiar with these rules.
In Pytest, a `"session"` scoped fixture cannot depend on a `"function"` scoped fixture.

## Overriding scopes

You may encounter situations where you don't want to make your users explicitly set the scope for each dependency.
For example, Spring defaults dependencies to the "singleton" scope.
Our approach is to give you a callback that gets information on the current context (the scopes passed to `Container.solve`, the current `DependantBase` and the scopes of all of it's sub-dependencies) where you can inject your own logic for determining the right scope.
Some examples of this include:

- A fixed default scope. You ignore all of the inputs and return a fixed value. This allows you to emulate Spring's behavior by returning a "singleton" scope or FastAPI's behavior by returning a "connection"/"request" scope.
- Try to assign the outermost valid scope. If the dependency depends on a `"request"` sub-dependency, you can't assign a `"singleton"` scope, so you assign the `"request"` scope. If there are no sub-dependencies or they all have the `"singleton"` scope, then you can assign the `"singleton"` scope.

Here is an example of the simpler fixed-default behavior:

```Python
--8<-- "docs_src/default_scope.py"
```

In this example we didn't provide a scope for `get_domain_from_env`, but `di` can see that it does not depend on anything with the `"request"` scope and so it gets assigned the `"singleton"` scope.
On the other hand `authorize` *does* depend on a `Request` object, so it gets the `"request"` scope.

[contextvars]: https://docs.python.org/3/library/contextvars.html
