# IFC/BIM Web Viewer

A browser-based viewer for IFC (Industry Foundation Classes) building models, built with Vue 3 and That Open Engine. Load an `.ifc` file from your computer, move around the model in 3D, browse its relations tree, and click any element to highlight it and see its properties.

I started this project to learn how IFC and BIM data work on the web. It's for developers and BIM learners who want a lightweight, open-source example of opening and inspecting IFC files in the browser, with no server or desktop software needed.

## Features

- **Open local IFC files** with a file picker; nothing is uploaded anywhere.
- **3D navigation** with a grid, camera controls and culling for smoother rendering.
- **Relations tree** panel with search, showing how the model's elements are organized.
- **Select and highlight** elements in 3D, zoom to the selection, and view the element's properties.
- **Loading progress** indicator for larger models.

Experimental tools in `src/components/` (working, but not yet available from the main UI):

- `IFCTileConverter.vue` converts an IFC file into geometry and property tiles for streaming.
- `AppIFCStreamingViewer.vue` loads those tiles, with a classifications tree.
- `IfcJsonExporter.vue` exports a model's properties to JSON.
- `IfcRelationsDownloader.vue` indexes a model and downloads its relations as JSON.

## Built with

- [Vue 3](https://vuejs.org/) with TypeScript (Vue CLI 5, webpack 5)
- [That Open Engine](https://thatopen.com/): `@thatopen/components`, `components-front`, `fragments`, `ui`, `ui-obc`
- [web-ifc](https://github.com/ThatOpen/engine_web-ifc) for parsing IFC files
- [three.js](https://threejs.org/) for 3D rendering

## Getting started

Requirements: Node.js 18 or later and npm. An internet connection is needed when loading a model, because the web-ifc WASM files are currently loaded from unpkg.

```bash
npm install
npm run serve     # development server at http://localhost:8080
```

Open the app, click **Load IFC File**, and pick an `.ifc` file. To try it right away, use `Building-Structural.ifc` from the project root.

Other commands:

```bash
npm run build     # production build in dist/
npm run lint      # lint and fix files
```

## Project structure

```
src/
  App.vue                         app shell (mounts the viewer)
  components/
    AppIFCViewer.vue              main viewer: loading, 3D scene, tree, selection
    AppIFCStreamingViewer.vue     viewer for streamed tiles (experimental)
    IFCTileConverter.vue          IFC to streaming tiles (experimental)
    IfcJsonExporter.vue           properties to JSON (experimental)
    IfcRelationsDownloader.vue    relations to JSON (experimental)
Building-Structural.ifc           sample model for testing
```

## Roadmap

Planned work is listed in [ROADMAP.md](ROADMAP.md).

## Sample data

`Building-Structural.ifc` is a public sample model ("ifc silly sample scene") published by [buildingSMART International](https://www.buildingsmart.org/) for testing IFC software. It contains no real project or personal data.

## License

[MIT](LICENSE)
