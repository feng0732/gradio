# @gradio/workflowcanvas

## 0.3.0

### Features

- [#13502](https://github.com/gradio-app/gradio/pull/13502) [`429faeb`](https://github.com/gradio-app/gradio/commit/429faeb643fb1afc1722c0f63fafa11603f2c87f) - Ensure every component dispatches a `change` event when its value changes.  Thanks @abidlabs!

### Fixes

- [#13484](https://github.com/gradio-app/gradio/pull/13484) [`87f42b9`](https://github.com/gradio-app/gradio/commit/87f42b9135004f416bfc71798c55eebf8120ef9c) - workflow: preserve dropdown, radio and checkbox inputs.  Thanks @hannahblair!

### Dependency updates

- @gradio/client@2.2.2

## 0.2.2

### Features

- [#13497](https://github.com/gradio-app/gradio/pull/13497) [`c26e797`](https://github.com/gradio-app/gradio/commit/c26e7975ede55b4c27d22379255de0b4e69ba3c6) - fix workflow canvas repo.url.  Thanks @pngwn!

## 0.2.1

### Fixes

- [#13487](https://github.com/gradio-app/gradio/pull/13487) [`61b9715`](https://github.com/gradio-app/gradio/commit/61b9715739936a73c03408eb8b435bf7833f524f) - workflow: tweak zerogpu toggle UX.  Thanks @hannahblair!

## 0.2.0

### Features

- [#13417](https://github.com/gradio-app/gradio/pull/13417) [`127400b`](https://github.com/gradio-app/gradio/commit/127400b01538faa0cf9d8a1ea7984c5ee05e66a0) - Add `gr.Workflow` and `gr.WorkflowCanvas` for building AI pipelines visually. `gr.Workflow` accepts `graph`, `bind`, and `edges` parameters to wire Python functions into a canvas, with automatic persistence to the graph JSON file.  Thanks @hannahblair!

## 0.1.0

### Features

- Initial release: Workflow Canvas component integrated into core Gradio