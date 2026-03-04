# gemini-cli-system
Add-ons for the gemini-cli

`system.md` is a customized system prompt for the Gemini CLI, derived from the original prompt, but with few fixes and optimizations.
Use it either by copying it into `.gemini/system.md` and running with `GEMINI_SYSTEM_MD=1`, or, if you want to enable it for the
project - you can create `.gemini/.env` with the content:

```
GEMINI_SYSTEM_MD=1
```

Alternatively, you can use GEMINI_SYSTEM_MD=\<file\>.

Original prompt can be extracted with:

```
GEMINI_WRITE_SYSTEM_MD=./system_original.md gemini
```

Version used as a basis is checked-in.
