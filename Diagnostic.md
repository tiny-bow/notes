# Diagnostic

With zig's logging and error handling systems, it makes sense to provide an
optional way of getting more information about errors in certain apis. We need a
org-wide consistent solution for this. Ideally, functions taking such a diagnostic input
would either log to the standard logFn or to the diagnostic, depending on wether
one was passed.
