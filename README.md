# Quoted Pair Parsing Confusion in python-multipart

**Researcher:** [@Loading886](https://github.com/Loading886)

## Summary

`python-multipart` misinterprets a quoted `Content-Disposition` parameter when a closing quotation mark follows an escaped backslash. The parser can then treat a semicolon inside a quoted `filename` as a parameter separator, changing the name and type of a multipart form part. The behavior was reproduced locally with `python-multipart` 0.0.32 and repository commit `212819d3b8f4c6b1e6f210232098681a5d8d8a56`.

## Reproduction

Run this example with `python-multipart==0.0.32`:

```python
from python_multipart.multipart import parse_options_header

header = b'form-data; name="safe\\\\"; filename="x;name=role;z"'
print(parse_options_header(header))
```

The resulting parameter map contains `name=b'role'` and no `filename`. Under [RFC 9110 quoted-pair rules](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.4), the two backslashes on the wire represent one literal backslash and the following quote closes the `name` value. The expected parameters are `name=b'safe\\'` and `filename=b'x;name=role;z'`.

In a complete multipart body using this header, the local `FormParser` emits a field named `role` rather than a file part named `safe\\`. A local Starlette `MultiPartParser` check with `python-multipart` 0.0.32 produced the same field-name interpretation.

## Security impact

An upstream component that follows quoted-string syntax may see a file part named `safe\\`, while an application using `python-multipart` sees a regular field named `role`. If the upstream component applies a security rule based on its interpretation, the application may receive a field the rule intended to reject. This depends on the deployment; no live application bypass was tested.

## Cause and remediation

In `_parseparam`, the parser estimates quote parity by subtracting the count of backslash-quote pairs from the total number of quotes. A quote preceded by an even number of backslashes is not escaped, but this counting method treats it as escaped. A parser that tracks quoted and escaped states in one pass avoids the incorrect split.

A local correction and regression test passed 20,000 generated valid quoted-parameter cases and 152 applicable project tests. This case uses ordinary quoted parameters and differs from [GHSA-vffw-93wf-4j4q](https://github.com/Kludex/python-multipart/security/advisories/GHSA-vffw-93wf-4j4q), which concerned extended `*` parameters.

[GHSA-5v7g-rp96-x2pf](https://github.com/Kludex/python-multipart/security/advisories/GHSA-5v7g-rp96-x2pf)
