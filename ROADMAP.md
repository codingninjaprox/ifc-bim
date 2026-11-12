# Roadmap

Planned work for the IFC/BIM Web Viewer. Estimates are rough.

## Viewer features

- [ ] Add navigation to switch between the viewer and the experimental tools (tile converter, streaming viewer, JSON exporter, relations downloader), which exist but aren't reachable from the UI (~3 h)
- [ ] Replace the raw JSON properties dump with a readable panel grouped by property sets and quantities (~4 h)
- [ ] Search and filter elements by name, IFC class or property value (~3 h)
- [ ] Show, hide and isolate elements by IFC class (walls, slabs, doors…) (~3 h)
- [ ] Measurement tools for length and area (~4 h)
- [ ] Clipping planes and section views (~3 h)
- [ ] Replace the VM-specific WebGL error message with general troubleshooting steps (~1 h)

## Reliability and performance

- [ ] Bundle the web-ifc WASM files locally instead of loading them from unpkg, and align the web-ifc version across components (currently 0.0.57 and 0.0.59) (~2 h)
- [ ] Code-split the 5.3 MB vendor bundle and lazy-load the tools (~3 h)
- [ ] Make the streaming pipeline work end to end for large models: convert, then load tiles in the streaming viewer (~5 h)
- [ ] Clear error messages for invalid or unsupported IFC files (~2 h)

## Code quality

- [ ] Use one component style across the project (currently a mix of `vue-facing-decorator` classes and `<script setup>`) (~2 h)
- [ ] Unit tests for property and relations helpers, plus a smoke test that loads the sample model (~4 h)
- [ ] GitHub Actions workflow that runs lint and build on every push (~1 h)

## Docs and release

- [ ] Screenshots or a short GIF in the README, plus a usage guide (~2 h)
- [ ] Deploy a live demo with the sample model (GitHub Pages or Netlify) (~2 h)
- [ ] First tagged release (v0.1.0) with a changelog (~1 h)
