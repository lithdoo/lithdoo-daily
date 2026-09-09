# 2026-09-09

## Done

- Designed, reviewed, and froze M13 Web Presentation across formal contracts, ADR, architecture, package, module, and implementation docs.
- Closed the presentation authority model as **Control Session/DataAuthority topology + per-subsystem Render Store facts → thin Projector → business-owned Custom Elements**.
- Implemented Web Presentation Config validation/preparation, ordered browser bootstrap, package-private presentation seam, Web Projector, and narrow PresentationResourceClient façade.
- Kept same-generation reconnect currentness per subsystem; fresh Session/generation retires old HTMLElement identity, while Control/Data transport loss preserves the last committed presentation.
- Added fail-closed unknown-tag preflight, Window-local structural failure latch, and Window-bounded resource lifetime/cancellation.
- Avoided `@loomrealm/presentation`, PresentationStore, component registry/loader, AssetManager, layer framework, service locator, and DOM rollback framework.
- Completed real Chromium qualification, including bootstrap ordering, Custom Element lifecycle/identity, reconnect, authority removal, structural failure, resource lifetime, and real Control/Data/Store → DOM vertical.
- Replaced production use of `snapshotForQualification()` with narrow Store `readPresentationFacts()` and tightened AbortSignal validation without introducing another Store/read-model layer.
- `npm run test:m13` passed in GitHub Actions on Node 20 and Node 24 with real Chromium.
- M13 is now **Implemented / Qualified / Closed**; next milestone is M14 `loom.map`.

## Records

- [LoomRealm M13 Web Presentation closure and system review](./loom-realm-m13-web-presentation-closure.md)

## Stable project update

- [LoomRealm current baseline](../../../projects/loom-realm/README.md)
- [M13 Web Presentation boundary](../../../projects/loom-realm/decisions/m13-web-presentation-boundary.md)
- [M13 Web Presentation qualified baseline](../../../projects/loom-realm/reviews/m13-web-presentation-qualified-baseline.md)

## Next

- M14 `loom.map`: first real business consumer of Frame / Input / Render / Content / Web Presentation.
- Use real map workload to validate author ergonomics and synchronous presentation cost; do not prebuild scheduler/SDK/framework without evidence.
- M15 closes real Desktop BrowserWindow composition, M16 PWA Runtime, and M17 PWA full E2E/equivalence.
