Componentization Plan: Grid/Parcel/Custom Search

- Goal: Move Grid, Parcel, and Custom dataset search into their own focused components, while preserving the shared UX for the search bar (auto-search, spinner, sticky header, transient status messages, and manual search/error handling).

- High-Level Approach:
    - Create three child components under `land-tract-geometry-select/`:
        - `land-survey-grid-geometry-search` (Survey Grid)
        - `land-parcel-geometry-search` (Parcel)
        - `land-custom-geometry-search` (Custom datasets)
    - Each child component owns its search input, auto/manual search streams, loading and status state, and rendering of its results table.
    - Parent `LandTractGeometrySelectComponent` becomes responsible for:
        - Selecting which child component to render based on the active type.
        - Passing in any dataset metadata (for Custom).
        - Receiving a single, normalized selection event and re-emitting `(geometrySelected)` or closing the dialog.

- Shared UX Requirements (kept consistent across children):
    - Sticky search bar: pinned at the top, with a neutral grey background to cover scrolled content.
    - Auto-search: debounce 750ms, 5s timeout, silent on error (only status text if needed).
    - Manual search: Enter and button trigger; 10s timeout; shows error message on failure.
    - Spinner: visible immediately when searching; shows “Searching” label; hides when complete.
    - Status messages: transient “No results found” or “Unknown server error” for ~2s after search completes.
    - Session/token: invalidate stale responses when the type/dataset changes.

- Child Component Contract (Inputs/Outputs):
    - Inputs:
        - `projectId: string` (optional unless API requires project scope)
    - Outputs:
        - `(selected)`: emits `{ type: string; label: string; geojson: object }`
        - `(searchLoading)`: emits `boolean` during loading if parent needs to react (optional)

- Suggested File Structure:
    - `land-grid-geometry-search/land-grid-geometry-search.component.ts|html`
    - `land-parcel-geometry-search/land-parcel-geometry-search.component.ts|html`
    - `land-custom-geometry-search/land-custom-geometry-search.component.ts|html`

- Implementation Sketch (per child component):
    - State Variables:
        - `search$ = new BehaviorSubject<string>('')`
        - `loading$ = new BehaviorSubject<boolean>(false)`
        - `statusMessage$ = new BehaviorSubject<string | null>(null)`
        - `error$ = new BehaviorSubject<string | null>(null)`
        - `results$ = new BehaviorSubject<any[]>([])`
        - `isLoaded$ = new BehaviorSubject<boolean>(false)`
    - Auto search pipeline:
        - `search$.pipe(filter(len>=3), debounceTime(750), tap(loading true + status ‘Searching’), apiCall().pipe(timeout(5s), catchError(() => setStatus('Unknown server error'); return of([])))).subscribe(updateResults)`
    - Manual search pipeline:
        - `manualSearch$.pipe(filter(len>=3), tap(error null + loading true + status ‘Searching’), apiCall().pipe(timeout(10s), catchError(() => error('Search failed or timed out'); setStatus('Unknown server error'); return of([])))).subscribe(updateResults)`
    - Helpers:
        - `setStatusTemporary(msg: string)` sets `statusMessage$` for ~2s.
        - On dataset/type change (via `@Input` changes): increment `session`, clear inputs/results/status/error/loading.
    - UI:
        - Sticky search bar div with input + Search button; label text underneath.
        - Centered spinner + “Searching” while `loading$`.
        - Below spinner: transient `statusMessage$`, then `p-table` when results present.
    - Selection:
        - Convert item shape to `{ type, label, geojson }` and emit `(selected)`.

- Parent Integration (LandTractGeometrySelectComponent):
    - Type selection remains in the left sidebar; on change, parent resets child by changing `ngIf`/keyed `*ngIf` to remount child (ensuring fresh session state).
    - `onSelect($event)` sets `selectedLocation` in parent and emits/finishes as it does today.

- Migration Steps:
    - Extract current grid logic into `land-grid-geometry-search` with shared UX behavior.
    - Extract current parcel logic into `land-parcel-geometry-search` with shared UX behavior.
    - Extract current custom logic into `land-custom-geometry-search`; accept `dataset` input and compute label/geometry parsing there.
    - Replace inline sections in `land-tract-geometry-select.component.html` with the new child components and wire outputs.
    - Keep existing public interface of the selector (projectId input and `(geometrySelected)` output) unchanged.

- Notes:
    - Keep child components presentation-only; reuse shared services (`MapApiService`, `ModuleLandApiService`) for IO.
    - For consistent styles, reuse the same CSS classes and inline styles used in the current selector for sticky header and spinner.
