# Interactive Geospatial Conservation Explorer

This web application displays an interactive geospatial data for environmental monitoring, allowing users to discover and explore satellite imagery dynamically.

## Setup and Installation

To run this project locally, follow these steps:

1.  **Clone the repository:**

    ```bash
    git clone <your-repository-url>
    cd <project-directory>
    ```

2.  **Install dependencies:**
    This project uses `npm` as the package manager.

    ```bash
    npm install
    ```

3.  **Set up environment variables:**
    Create a `.env` file in the root of the project by copying the example file:

    ```bash
    cp .env.example .env
    ```

    Open the `.env` file and add your free API key from [MapTiler](https://www.maptiler.com/cloud/).

    ```env
    PUBLIC_MAPTILER_API_KEY="your_maptiler_api_key_goes_here"
    ```

4.  **Run the development server:**
    ```bash
    npm run dev
    ```
    The application will be available at `http://localhost:5173`.

## Architecture Overview

The application is built on a modern, reactive frontend stack, prioritizing simplicity and maintainability.

- **Framework:** **SvelteKit** was used for its speed, simplicity, and powerful reactivity model, which is ideal for managing the complex state of an interactive map.
- **Mapping Library:** **MapLibre GL JS** serves as the core mapping engine, chosen for its high performance with WebGL and its excellent compatibility with standard geospatial data formats.
- **Styling:** **TailwindCSS** provides a utility-first approach for rapid, consistent, and maintainable styling of UI components.
- **Dynamic Tiling:** Instead of serving raster data directly, the application uses the public **TiTiler** API (`titiler.xyz`). By constructing a `tilejson.json` URL, we dynamically get both the tile endpoints and the precise bounding box for any given Cloud-Optimized GeoTIFF (COG). This is a robust, server-driven approach that simplifies the client-side code.
- **Data Discovery:** The "Find New Satellite Image" feature uses the **Microsoft Planetary Computer STAC API**. This allows for live queries against a massive catalog of real-world satellite data.
- **Authentication:** Asset URLs from the Planetary Computer are protected. The application implements the required authentication flow by sending the unsigned asset URL to a dedicated `/sign` endpoint to receive a temporary, signed URL that can be accessed by TiTiler.

The entire application logic is encapsulated within a single Svelte component (`+page.svelte`) for this assignment, but the UI panel and map logic could be easily refactored into separate components for a larger application.

## Features Prioritized (and Why)

The development followed the priorities outlined in the assignment brief, with a focus on creating a solid foundation.

1.  **Core Features (Priority 1):**
    - **Map Foundation & GeoJSON:** The first step was to get a MapLibre map running and display a `GeoJSON` boundary layer. This established the project's foundation.
    - **Raster Data (COG):** Next, we focused on overlaying a static COG file. This led to the architectural decision to use the TiTiler `tilejson.json` endpoint, which proved to be the most reliable method.
    - **STAC Integration:** Finally, we integrated the Planetary Computer STAC API to dynamically discover and display new imagery. This completed the core data pipeline of the application.

2.  **Bonus Features (Priority 2):**
    - **UI Controls:** After the core features were working flawlessly, we prioritized building a comprehensive UI panel. This was chosen because it adds the most value to the user experience, making the application truly interactive and demonstrating the power of the underlying data layers.
    - **Performance:** Performance was a key consideration and was addressed through several architectural choices:
      - **Efficient Data Loading:** The core decision was to use a tile server (TiTiler) to process Cloud-Optimized GeoTIFFs. This is highly performant as the browser only ever downloads small, necessary map tiles for the current view, not the entire source image, which could be gigabytes in size.
      - **Smooth Interactions:** MapLibre's WebGL-based rendering ensures smooth panning and zooming. All view changes are handled with animated transitions (`fitBounds`, `flyTo`), and the `maxZoom` parameter is used to prevent jarringly large zoom-outs, providing a more fluid user experience.
      - **Responsive UI:** The UI is built with TailwindCSS and is fully responsive. Asynchronous actions provide immediate user feedback with loading states, preventing the UI from feeling frozen during data fetching.
    - **Backend Proxy:** The decision was made **not** to implement the FastAPI backend proxy. While a valuable feature for a production system, building a high-quality, fully-featured and performant frontend was a better use of time and more effectively demonstrated the required software development approach.

## Testing Approach

Given the time constraints, a formal testing suite was not implemented. However, the application was subjected to a rigorous **manual, user-centric testing process**:

- **Initialization:** Ensured the map loaded correctly with the initial Austria boundary and demo COG.
- **Core Functionality:** The "Find New Satellite Image" button was tested repeatedly to ensure the search -> sign -> display data flow was reliable and that images were being randomized.
- **UI Controls:** Every control in the UI panel (toggles, sliders, basemap switcher) was tested to confirm it correctly manipulated the map's state and that the state persisted across actions (e.g., opacity settings are remembered after switching basemaps).
- **Error Handling:** The `try...catch` block in the image search function was tested by simulating network errors to ensure the `alert` fallback worked as expected.
- **Cross-Browser Check:** A quick check was performed on modern versions of Chrome and Firefox.
- **Console Monitoring:** The browser's developer console was monitored throughout testing to catch and resolve warnings and errors.

## Challenges & Future Improvements

- **Challenges:**
  - The most significant challenge was reliably fetching the bounding box for a COG from the client. Initial attempts using `geotiff.js` failed due to coordinate projection issues. Several attempts to use TiTiler's `info` endpoint also failed due to incorrect or unreliable public API endpoints. The breakthrough came from using the `/WebMercatorQuad/tilejson.json` endpoint, which provided both the bounds and tiles in a single, robust API call.
  - Discovering and correctly implementing the asset signing flow for the Planetary Computer was another key challenge that required careful reading of their API documentation.

- **Future Improvements:**
  - **Backend Proxy:** The most logical next step would be to build the FastAPI + TiTiler backend. This would make the application self-contained and remove the reliance on public APIs, increasing reliability.
  - **Formal Testing:** Introduce a testing framework like Vitest for unit tests on data-fetching functions and Playwright for end-to-end tests on user interactions.
  - **State Management:** For a larger application, Svelte's built-in stores could be used to manage the map and layer state more formally, allowing different components to interact with it without passing props.
  - **Enhanced UI/UX:** Replace `alert()` messages with more elegant toast notifications for a smoother user experience.
