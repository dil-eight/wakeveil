# Scripts

Install, uninstall and recovery scripts for the macOS authorization component, plus relay deployment and load-test scripts later.

Every script that changes system state:

- requires explicit administrator authorization;
- backs up what it replaces;
- has a matching undo.

See [`docs/spec.md` §1](../docs/spec.md#1-first-milestone-validate-macos-unlocking) and [§13](../docs/spec.md#13-resource-budget).
