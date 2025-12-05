# Sensorium Webapp

A Django-based web application for visualizing and exploring mouse neural data, including videos, neural responses, behavior, and pupil center data.

## Launch Instructions

### Prerequisites

1. **Python 3.10+** installed
2. **Virtual environment** activated (if using one)
3. **Dependencies** installed (see below)

### Step 1: Activate Virtual Environment

If you're using a virtual environment:

```bash
cd /home/l11/Desktop/Sensorium
source venv/bin/activate
```

### Step 2: Verify Dependencies

Ensure Django and required packages are installed:

```bash
python -c "import django; print('Django version:', django.get_version())"
```

If Django is not installed, install it:

```bash
pip install django numpy opencv-python plotly
```

Or if using `uv`:

```bash
uv pip install django numpy opencv-python plotly
```

### Step 3: Run Database Migrations (if needed)

For a fresh setup, you may need to run migrations:

```bash
python scripts/manage.py migrate
```

Note: This webapp doesn't use Django's database models, so migrations are optional.

### Step 4: Start the Development Server

You can run the server from the project root:

```bash
python scripts/manage.py runserver 0.0.0.0:8050
```

Or navigate to the scripts directory first:

```bash
cd scripts
python manage.py runserver 0.0.0.0:8050
```

The server will start and display:
```
Watching for file changes with StatReloader
Starting development server at http://0.0.0.0:8050/
Quit the server with CONTROL-C.
```

### Step 5: Access the Webapp

Open your web browser and navigate to:

```
http://localhost:8050
```

or

```
http://0.0.0.0:8050
```

### Stopping the Server

Press `Ctrl+C` in the terminal where the server is running.

### Troubleshooting

**Port already in use:**
```bash
# Kill any process using port 8050
pkill -f "manage.py runserver"
# Or use a different port
python scripts/manage.py runserver 0.0.0.0:8051
```

**Module not found errors:**
- Ensure virtual environment is activated
- Verify all dependencies are installed
- Check that `webapp` directory exists (contains all webapp files including data loader utilities)

**Static files not loading:**
- Django development server serves static files automatically
- Check browser console for 404 errors
- Verify `webapp/static/webapp/` directory structure exists

**Video not playing:**
- Check browser console for video loading errors
- Verify video conversion is working (requires OpenCV)
- Ensure video data is being returned from API endpoint `/api/mice/<mouse_id>/videos/<video_id>/video/`

**3D plot not displaying:**
- Check browser console for Plotly.js errors
- Verify cell coordinates API endpoint returns data: `/api/mice/<mouse_id>/cell_coordinates/`
- Ensure Plotly.js CDN is loading (check Network tab)

**Plots not showing:**
- Check browser console for JavaScript errors
- Verify API endpoints return data: `/api/mice/<mouse_id>/videos/<video_id>/plot/<data_type>/`
- Check that video IDs match between API responses and plot generation

## Webapp Architecture

### Overview

The Sensorium webapp is a **Django-based single-page application** that provides an interactive interface for exploring mouse neural data. It uses **client-side rendering** with **Plotly.js** for visualizations and **AJAX** for dynamic data loading.

### Technology Stack

- **Backend**: Django 5.2.9 (Python web framework)
- **Frontend**: HTML5, CSS3, JavaScript (vanilla JS, no frameworks)
- **Visualization**: Plotly.js (client-side plotting library)
- **Data Processing**: NumPy, OpenCV (Python backend)
- **Architecture Pattern**: Server-side rendering with AJAX API endpoints

### Project Structure

```
Sensorium/
├── webapp/                    # Single directory containing ALL webapp files
│   ├── settings.py            # Django settings (apps, static files, etc.)
│   ├── project_urls.py        # Main URL routing (root URLconf)
│   ├── wsgi.py                # WSGI configuration (for deployment)
│   ├── asgi.py                # ASGI configuration (for async deployment)
│   ├── views.py               # View functions + API endpoints
│   ├── urls.py                # App-specific URL routing
│   ├── apps.py                # App configuration
│   ├── data_loader.py         # MouseDataManager class
│   ├── loader.py              # SensoriumSession class
│   ├── db.sqlite3             # SQLite database (if used)
│   ├── utils/
│   │   └── video_converter.py # Video conversion utilities
│   ├── templates/
│   │   └── webapp/
│   │       └── index.html     # Main HTML template
│   └── static/
│       └── webapp/
│           ├── css/
│           │   └── style.css  # Stylesheet
│           └── js/
│               ├── main.js    # Main JavaScript (AJAX, event handlers)
│               └── plotting.js # Plotly.js plotting functions
│
├── scripts/                   # Scripts and notebooks
│   ├── manage.py              # Django management script
│   ├── data_exploration.ipynb # Data exploration notebook
│   ├── experiments.ipynb      # Experiments notebook
│   └── create_combined_metadata.py # Metadata creation script
│
├── project_data/              # Mouse data directories
├── results/                   # Processed metadata JSON files
└── extra/                     # Old Dash files (can be deleted)
```

### Data Flow

#### 1. Initial Page Load

```
Browser Request → Django View (index) → Template Rendering → HTML Response
```

- User navigates to `http://localhost:8050`
- Django `index()` view in `webapp/views.py` loads:
  - Available mice from `MouseDataManager`
  - Default mouse, video, and neuron
- Template `webapp/templates/webapp/index.html` renders with default values
- HTML includes Plotly.js CDN and static files (CSS, JS)

#### 2. Client-Side Initialization

```javascript
DOMContentLoaded → initializeApp() → AJAX Calls → Plot Rendering
```

- JavaScript in `main.js` detects page load
- Calls `initializeApp()` with default values
- Makes AJAX requests to:
  - `/api/mice/<mouse_id>/videos/` - Load video dropdown
  - `/api/mice/<mouse_id>/cell_coordinates/` - Load 3D plot
  - `/api/mice/<mouse_id>/videos/<video_id>/info/` - Load metadata
  - `/api/mice/<mouse_id>/videos/<video_id>/plot/<data_type>/` - Load plot data

#### 3. User Interactions

**Mouse Selection:**
```
User selects mouse → AJAX call → Update dropdowns → Load 3D plot → Clear grids
```

**Video Selection:**
```
User selects video → AJAX calls → Update video player → Load metadata → Load all grids
```

**Neuron Selection:**
```
User selects neuron → AJAX call → Update only neuron responses grid
```

### Key Components

#### Backend (Django)

**`webapp/views.py`** - Contains:
- `index()`: Main view that renders the template
- `api_get_videos()`: Returns representative videos for a mouse
- `api_get_video_info()`: Returns video metadata (valid frames, equivalent videos)
- `api_get_neurons()`: Returns number of neurons
- `api_get_plot_data()`: Returns plot data (responses/behavior/pupil_center) as JSON
- `api_get_cell_coordinates()`: Returns 3D cell coordinates
- `api_get_video_base64()`: Converts numpy video array to base64 for HTML5 playback

**`MouseDataManager`** (from `webapp/data_loader.py`):
- Discovers available mice from `project_data/` directory
- Loads metadata from `results/` JSON files
- Provides methods to retrieve:
  - Representative videos
  - Equivalent videos
  - Video/response/behavior/pupil data
  - Cell motor coordinates

#### Frontend (HTML/CSS/JavaScript)

**`index.html`** - Main template structure:
- Top section: Mouse ID dropdown (centered)
- Three-column layout:
  - Left: Video dropdown + HTML5 video player
  - Middle: 3D Plotly plot (cell motor coordinates)
  - Right: Metadata display
- Three data sections:
  - Neuron Responses (with neuron dropdown)
  - Behavior
  - Pupil Center
- Each section uses expandable canvas (horizontal scrolling)

**`style.css`** - Styling:
- Professional, clean design
- CSS Grid for expandable canvas
- Fixed plot sizes (400px × 300px)
- Horizontal scrolling for many equivalent videos
- Responsive design considerations

**`main.js`** - Main JavaScript logic:
- Event handlers for dropdowns (mouse, video, neuron selection)
- AJAX functions for API calls
- Grid generation functions (properly appends DOM elements for Plotly rendering)
- Dynamic plot updates
- Video player creation with error handling
- 3D plot initialization with proper container dimensions and centering

**`plotting.js`** - Plotly.js functions:
- `plotSingleNeuronResponse()`: Single neuron trace plot
  - Handles (n_neurons, n_frames) or (n_frames, n_neurons) orientation
  - Extracts specific neuron or computes mean
- `plotSingleBehavior()`: Dual y-axis (running speed + pupil dilation)
  - Handles (2, n_frames) or (n_frames, 2) data shapes
  - Robust shape detection with fallback handling
- `plotSinglePupilCenter()`: Dual y-axis (horizontal + vertical position)
  - Handles (2, n_frames) or (n_frames, 2) data shapes
  - Robust shape detection with fallback handling
- `plotCellCoordinates3D()`: 3D scatter plot
  - Requires proper container dimensions (400px height)
  - Uses Plotly 3D scene with interactive controls

### Expandable Canvas Implementation

The expandable canvas allows the window to adapt to the number of plots rather than plots adapting to the window:

**CSS Grid Approach:**

```css
.data-grid-container {
    width: 100%;
    overflow-x: auto;  /* Horizontal scrollbar when needed */
}

.data-grid {
    display: grid;
    grid-auto-flow: column;  /* Columns flow horizontally */
    grid-auto-columns: minmax(400px, 400px);  /* Fixed column width */
    width: max-content;  /* Container expands to content */
}
```

**How it works:**
1. Each plot cell has fixed dimensions (400px × 300px)
2. Grid container uses `grid-auto-flow: column` to create columns dynamically
3. Container width is `max-content`, so it expands based on number of columns
4. Parent container has `overflow-x: auto`, creating horizontal scrollbar when content exceeds viewport width

### API Endpoints

All endpoints return JSON:

- `GET /api/mice/<mouse_id>/videos/`
  - Returns: `{"videos": ["0", "3", "6", ...]}`

- `GET /api/mice/<mouse_id>/videos/<video_id>/info/`
  - Returns: `{"video_valid_frames": 100, "number_equivalent_videos": 5, "equivalent_videos": ["12", "14", ...]}`

- `GET /api/mice/<mouse_id>/videos/<video_id>/neurons/`
  - Returns: `{"num_neurons": 8000}`

- `GET /api/mice/<mouse_id>/videos/<video_id>/plot/<data_type>/`
  - `data_type`: `"responses"`, `"behavior"`, or `"pupil_center"`
  - Returns: `{"video_ids": ["0", "12", "14"], "plot_data": {"0": [[...]], "12": [[...]]}}`

- `GET /api/mice/<mouse_id>/cell_coordinates/`
  - Returns: `{"coordinates": [[x, y, z], ...]}`

- `GET /api/mice/<mouse_id>/videos/<video_id>/video/`
  - Returns: `{"video_data": "base64_encoded_video_string"}`

### Data Loading Strategy

**Lazy Loading:**
- Data is loaded on-demand via AJAX
- Only requested data is fetched from disk
- Reduces initial page load time

**Caching:**
- `MouseDataManager` caches loaded sessions and metadata
- Reduces redundant file I/O operations
- Improves performance for repeated requests

**Data Format:**
- Videos: NumPy arrays converted to base64 MP4 (via OpenCV)
- Plot data: NumPy arrays converted to JSON lists
  - Responses: (n_neurons, n_frames) format
  - Behavior: (2, n_frames) where [0]=running speed, [1]=pupil dilation
  - Pupil Center: (2, n_frames) where [0]=horizontal, [1]=vertical
- Metadata: JSON files parsed once per mouse
- Cell Coordinates: (N, 3) numpy array → JSON list of [x, y, z] coordinates

### Key Implementation Details

**Grid Plot Rendering:**
- Plots are created by appending DOM elements (not innerHTML copy)
- Each plot cell gets a unique ID: `plot-{dataType}-{videoId}`
- Plotly plots are rendered after DOM elements exist
- Fixed plot sizes (400px × 300px) ensure consistent layout

**Video Conversion:**
- NumPy video arrays are converted server-side using imageio with ffmpeg
- Videos are encoded as MP4 with H.264 codec (browser-compatible)
- Falls back to OpenCV if imageio is not available
- Base64 encoding for HTML5 video element compatibility
- Error handling for conversion failures

**3D Plot Rendering:**
- Requires explicit container dimensions (500px height)
- Plotly.js renders interactive 3D scene with camera controls
- Properly centered in panel using CSS flexbox and Plotly layout settings
- Uses cube aspect mode for equal axis scaling
- Container cleared before rendering to prevent overlap

### Design Decisions

1. **Django over Dash**: User requested Django specifically for more control over architecture
2. **Client-side Plotly**: Plots render in browser for interactivity (zoom, pan, hover)
3. **AJAX API**: Separates data loading from page rendering for better UX
4. **Fixed Plot Sizes**: Ensures consistent layout regardless of number of equivalent videos (400px × 400px for grid plots, 500px height for 3D plot)
5. **Expandable Canvas**: Horizontal scrolling allows viewing many videos without cramping
6. **No Database**: Uses file system directly (no Django models needed)
7. **DOM Element Appending**: Plots created by appending elements (not innerHTML) to preserve Plotly bindings
8. **Robust Data Shape Detection**: Plotting functions handle multiple data orientations with fallbacks
9. **Explicit Container Dimensions**: 3D plots require fixed dimensions to render correctly
10. **Error Handling**: Comprehensive error handling for video conversion, API calls, and plot rendering

### Known Issues & Solutions

**Issue: Video not playing**
- **Solution**: Check browser console for errors. Verify OpenCV is installed and video conversion succeeds. Ensure base64 data URI format is correct.

**Issue: 3D plot overlapping with metadata**
- **Solution**: Fixed with CSS Grid layout (`grid-template-columns: 1fr 2fr 0.8fr`) and explicit container dimensions. Ensure plot container has `height: 500px` and proper centering.

**Issue: 3D plot not centered**
- **Solution**: Fixed by removing flexbox interference, setting all margins/padding to 0, and using Plotly's proper layout settings with cube aspect mode.

**Issue: Video format not supported by browser**
- **Solution**: Switched from OpenCV's mp4v codec to imageio with ffmpeg H.264 encoding for browser compatibility.

**Issue: Plots not displaying**
- **Solution**: Fixed by properly appending DOM elements instead of using innerHTML. Ensure Plotly.js loads before plot rendering. Check API responses match expected data shapes.

**Issue: Behavior/Pupil data showing wrong values**
- **Solution**: Fixed data shape detection in plotting functions. Functions now handle both (2, n_frames) and (n_frames, 2) orientations with proper fallbacks.

### Future Enhancements

Potential improvements:
- Add caching layer (Redis) for frequently accessed data
- Implement WebSocket for real-time updates
- Add data export functionality
- Implement user authentication (if needed)
- Add filtering/search capabilities
- Optimize video loading (lazy load, thumbnails)
- Add loading indicators for better UX
- Implement plot export (PNG, SVG)
- Add keyboard shortcuts for navigation
