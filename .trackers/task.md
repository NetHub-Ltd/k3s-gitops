# Task Tracker

**Branch:** `fix/image-policy-bounded-semver`  
**Base:** `main`

## Goal
Bound tawala-api ImagePolicy to 0.0.x and document multi-app image automation pattern.

## Approved scope
- ImagePolicy range `>=0.0.0 <0.1.0`
- exclusionList for main/latest/sig
- README section 5: versioning contract, template, checklist
- No manual deployment image pin

## Done
- [x] image-tawala-api.yaml policy + exclusions
- [x] README section 5 rewrite

## Remaining
- [ ] PR merge + cluster verify elected tag is 0.0.40
