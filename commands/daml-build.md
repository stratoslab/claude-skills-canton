---
description: Compile DAML source to a DAR archive. Clean build if needed.
---

# Build DAML Project

1. Run `daml build` in the project directory
2. If build fails, analyze the error:
   - **Type errors**: Fix DAML source code
   - **Missing dependencies**: Check `daml.yaml` dependencies
   - **SDK version mismatch**: Check `sdk-version` in `daml.yaml`
3. Report the output DAR path (`.daml/dist/<name>-<version>.dar`)
4. If `--clean` was requested, run `daml clean && daml build`
