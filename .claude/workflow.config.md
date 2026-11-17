# Workflow Config

## Linear
- Team key: COM
- Team ID: f1e10efe-5164-4d71-a8b2-b16d93f1447f
- Default label: EspFrame Public

## Repository
- Base branch: main
- Branch prefix: feature/
- Git host: github
- Tracker: linear

## Build
<!-- check:fast / check:pr fail on stale product-contract checks from the original project.
     Use the CI-equivalent commands until COM-661 fixes them, then switch back. -->
- Lint: npm run webserver:typecheck
- Build: npm run webserver:build
- Test: npm run check:ci

## Open Code Review Gate
- Enabled: no
