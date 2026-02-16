---
title: "Building Scanlateanator: A Web-Based Manga Scanlation Editor"
description: A detailed look at my coding journey to create this HTML, CSS, and Vanilla JavaScript website.
date: 2026-02-16 14:55:00 -5000
category: [Blog Post]
tags: [HTML, CSS, JavaScript, coding, web design]
---
[Scanlateanator](https://pranjal-adhikari.github.io/personal-projects/scanlateanator) is a specialized web application I designed for basic manga scanlation. Manga scanlation is the process of translating and editing manga pages for different languages. This web app is designed as a tool to aid up-and-coming beginners in scanlating... **without** having to download or pay for an application. It includes brush features, text management with custom font uploads, layering, multi-page management, and an undo/redo history on a per-page basis. 

This article discusses the technical decisions, challenges, and solutions that I faced/made as I created this single-page web application.

## The Core Problem

Manga scanlation involves several tedious steps: removing original text from speech bubbles (typically by brushing over them in white), adding translated text with proper formatting (often with unique fonts typically not offered for free by most web-based image editors), managing multiple pages simultaneously, constantly reworking edits, and then exporting the final results in a filetype that is suitable for the user's needs. Typical scanlation requires complex desktop software (such as Adobe Photoshop and GIMP) with steep learning curves. My goal was to create something simpler, faster, and far more accessible directly in the browser.

## Architecture Overview

The application is built as a pure client-side web app with three main files:

- **index.html** (32KB): For the structure
- **script.js** (67KB): For the logic and canvas manipulation
- **styles.css** (24KB): Visual styling with a dark VSCode-inspired theme

This vanilla JavaScript approach avoids framework overhead and keeps the entire application lightweight and fast. There are no build processes, because this is vanilla JS with HTML+CSS for theming and structure. Due to this, you can simply open it in a browser and start editing.

## The Canvas System

At the heart of Scanlateanator is a multi-layered canvas system that separates different parts of image editing:

### Layer Architecture

The application uses two HTML5 canvas elements stacked on top of each other:

1. **Image Canvas**: This displays the original manga page (it's never modified directly)
2. **Edit Canvas**: This stores brush and eraser strokes with transparency

Text elements are implemented as HTML divs with absolute positioning, which allows for rich formatting (italics, boldening, colors, font size, etc.) and easy manipulation without having to deal with complex canvas text rendering.

This is also very important for performance. When a user paints with the brush tool, only the edit canvas is modified. When they add text, it's rendered as a DOM element that can be styled with CSS and easily repositioned. I've found a hybrid approach like this provides the best of both worlds rather than one over the other.

### Canvas Context Configs

While coding, I ran into an issue with optimizing for the browser. So, the code uses specific canvas context settings that are optimized for frequent read/write operations:

```javascript
const ctxEdit = editCanvas.getContext("2d", { 
  willReadFrequently: true, 
  desynchronized: true 
});
```

The `willReadFrequently` flag tells the browser that this canvas will be read often (for undo/redo operations), allowing it to optimize accordingly. The `desynchronized` flag paves the way for better performance by allowing the canvas to render independently of the page compositing.

## Drawing Engine (it's not really)

### Brush and Eraser Implementation

Simply put, the brush tool uses a line-drawing algorithm that creates additions between mouse positions to create smooth strokes even when users have fast movements:

```javascript
// For each mouse move event, draw a line from the last position to current position
// This prevents gaps in fast strokes
```

The eraser works by setting `globalCompositeOperation` to `'destination-out'`, which removes pixels instead of adding them. This is a fun (and less memory-intesive!) way to use the canvas compositing modes because it avoids having to track what needs to be erased.

### Stroke Optimization

One of the trickiest performance challenges was the undo/redo system. I faced so many problems with this, ranging from forgetting a bracket in my script which took down the whole operation to having to completely rework my undo/redo stacking system. Storing the entire canvas state for every action would quickly consume a lot of memory, especially for high-resolution images (ex. 2000x3000 pages). Throughout Scanlateanator's development, I was fretting over memory consumption left and right, constantly looking at Safari's Page Resources timeline that kept telling me—to an insanely aggravating extent—how much "high" CPU usage there was.

Optimization was my savior, optimization was my enemy. It was the path at the end of the tunnel, yet it was also the walls of the never-ending tunnel. I was breathing, eating, and dreaming about optimization. Thus, while in that state and scrolling around Stack Overflow and Reddit threads, I stumbled onto an interesting solution: **region-based state storage**. Instead of saving the entire canvas, the app calculates the bounding box of the affected area and stores only that region (!!!):

```javascript
const getStrokeBounds = (points, size, padding) => {
  // Calculate minimum bounding rectangle of stroke points
  // Add padding for brush size
  // Return a compact region instead of the full canvas
};
```

I've found that this approach reduces memory usage by a large sum for typical strokes that only affect a small portion of the canvas.

### WebP Compression

Additionally, to further optimize memory usage, the application compresses stored regions to WebP format (or PNG as fallback if the browser does not support WebP):

```javascript
const compressRegionToBlob = async (imageData) => {
  const tempCanvas = document.createElement('canvas');
  tempCanvas.width = imageData.width;
  tempCanvas.height = imageData.height;
  const tempCtx = tempCanvas.getContext('2d');
  tempCtx.putImageData(imageData, 0, 0);
  
  return new Promise(resolve => {
    tempCanvas.toBlob(resolve, 'image/webp', 0.85);
  });
};
```

There is an issue, though. By my logic, since WebP is a more modern filetype for images, it should show a more notice file size reduction compared to raw PNG storage while maintaining visual quality... right? Alas, I've found that this supposed benefit doesn't exist for smaller page edits, which I find quite odd. 

The app detects WebP support at startup and falls back to PNG for older browsers (which is also an interesting piece of code, because I believe I've coded it correctly, but it keeps falling back on PNG, even on Chrome).

## Text System

Ah, now we've gotten to the best boxes which are text boxes: one of the most complex features because they need to be interactive, editable, fluid, and exportable. I have pulled out a few hairs on this.

### Text Box Structure

Each text box is a positioned div with multiple child elements:

- **Content div**: This contains the actual editable text
- **Handles**: There are eight resize handles (corners and edges) plus a rotation handle
- **Transform layer**: This manages rotation without affecting the text layout

### Text Manipulation

The text system supports:

- **Double-click editing**
- **Drag to move**:
- **Resize**
- **Rotate**
- **Formatting**

The rotation implementation was particularly interesting. Instead of rotating the div directly (which would make the text hard to read and edit), the system:

1. Stores rotation as a data attribute
2. Applies rotation via CSS transform only during export
3. Keeps text upright during editing for usability

### Text Rendering for Export

When exporting, text must be rendered onto the canvas, involving:

1. Creating a temporary canvas for each text box
2. Drawing the text with all formatting (stroke, fill, font)
3. Rotating the temporary canvas by the stored angle
4. Compositing it onto the final export canvas at the correct position

The stroke implementation draws the text multiple times in a circle to create a decently-consistent outline effect. I coded that in, because HTML5 canvas doesn't have built-in text stroke support that looks good. However, I still find issues with the stroke feature, because after 4px, the corners begin to show, reducing the consistency.

## Undo/Redo System

The undo system maintains two stacks: `undoStack` and `redoStack` where each state captures:

- The compressed edit canvas region
- All text boxes with complete state, including position, size, rotation, content, and formatting
- And the action type for labeling in the visual history timeline

### Smart State Management

The system limits history to 20 steps to prevent memory issues with large files. When a new action occurs:

1. The current state is captured and compressed
2. It's pushed onto the undo stack
3. The redo stack is cleared, meaning it's impossible to redo after making a new change
4. If the stack exceeds 20 items, the oldest is removed

Keyboard shortcuts (Ctrl+Z, Ctrl+Y) trigger the appropriate history navigation.

### History Timeline UI

The left sidebar shows a visual history timeline where users can click any previous state to jump directly to it. I was inspired by the Google Docs revision history feature. I find this is more powerful than traditional linear undo/redo, because you can skip multiple steps instantly.

## Multi-Page Management

The intended users for this application often work on multiple manga pages in a single session. The page system maintains an array of page objects, each containing:

- The original image
- Canvas state (the edit layer)
- Text boxes (another layer)
- Undo/redo history
- Zoom and pan state

When switching pages, the application:

1. Serializes the current page state
2. Stores it in the pages array
3. Loads the target page state
4. Redraws all canvases and text boxes
5. Updates all UI elements

This creates an illusion of working with multiple different pages when users are actually just swapping out the active state.

## Zoom and Pan

The zoom system was one of the last features I incorporated, because I had predicted beforehand that there would be many bugs with it. Nonetheless, after implementing it, the zoom and fan have several features:

### Zoom Controls

- **Ctrl + Mouse Wheel**: Zoom towards mouse cursor position
- **+/- Keys**: Zoom in/out centered on viewport
- **0 Key**: Reset to 100%
- **Fit Button**: Auto-zoom to fit image in viewport

The zoom-to-cursor implementation was more math than anything. When zooming towards the mouse, the pan offset must be adjusted so the point under the cursor stays stationary:

```javascript
// Calculate zoom delta
const zoomDelta = newZoom - oldZoom;

// Adjust pan to keep mouse position constant
// This requied converting mouse coordinates to canvas space
panX = (panX - mouseX) * (newZoom / oldZoom) + mouseX;
panY = (panY - mouseY) * (newZoom / oldZoom) + mouseY;
```

### Pan Implementation

Panning was actually quite simple to implement. It is achieved by:

1. Holding spacebar or middle mouse button
2. Dragging to change pan offset
3. Updating canvas wrapper position via CSS transform

All drawing operations account for zoom and pan by transforming coordinates before rendering them. This makes sure brush strokes appear at the correct visual location regardless of zoom level (... a bug that had to be tweaked!!!!!).

## Export System

Exporting combines all layers into a single flattened image:

### Export Process

1. Create a temporary canvas at full resolution
2. Draw the original image
3. Draw the edit layer (brush strokes)
4. Render each text box with proper rotation and formatting
5. Convert to WebP or PNG
6. Download

### Progress Animation

What is one thing that all professional applications have? A progress bar! The export includes a loading overlay with animated progress bar. Of course, since canvas operations are synchronous and don't emit progress events, the system... simulates... progress:

```javascript
await animateProgress(0, 30, 300, 'Preparing canvas...');
// ... do actual work ...
await animateProgress(30, 60, 200, 'Rendering text...');
// ... render text ...
await animateProgress(60, 100, 200, 'Finalizing export...');
```

This provides user feedback even though the actual operations are instantaneous. Honestly, I think the UI looks more polished with this in it than if it were removed.

### Batch Export

The "Export All Pages" feature creates a ZIP file containing all pages:

1. For each page, generate an export canvas
2. Convert each to blob
3. Use JSZip library to create archive
4. Download the ZIP file

## The Responsive Design Idea

While coding Scanlateanator, I was completely swamped by the JavaScript logic. That meant any notion of responsive web design was a z-index: -2147483648 priority lol. Eventually, I realized... I don't need to make the editor responsive: NO ONE ELSE DOES! So, rather than cramming the complex canvas editor onto mobile devices, and practically building a whole new app, the application takes a different approach: it detects screen size and shows a landing page UI on small tablet/mobile devices.

### Mobile Landing Page

The mobile experience is a landing page that:

- Explains what Scanlateanator does
- Shows why it requires a desktop/laptop/large screen
- Lists features and capabilities
- Displays screenshots of the UIX
- Encourages users to visit on a desktop

I made this decision for my own sake, as well as to prioritize feature accessibility. While that sounds contradictory, it makes sense in my mind: I don't want to give users a badly-optimized time on mobile because buttons are too small, the navigation bar decided to stack, finger movements aren't being tracked, they've having a hard time navigating, and so on. The tool genuinely requires a mouse and keyboard to be usable, so attempting to shoehorn it onto touch devices would result in a frustrating experience.

The media query switches at 1024px width:

```css
@media (max-width: 1024px) {
  #desktopApp { display: none !important; }
  #mobileLanding { display: block; }
}
```

## Performance Optimizations

Several techniques keep the application functioning as efficiently as I can possibly code it:

### Image Size Limits

Images over 4096px in either dimension are automatically downscaled. I realize most manga pages are not this resolution—typically far smaller—but I want to keep everything open to possibility. The downscaling at 4096px prevents memory issues on large manga pages while maintaining sufficient resolution for editing.

### Debouncing Text Modification Updates

Text modifications trigger undo state saves, but if saved on every keystroke, the undo stack (and visual history timeline) would fill with intermediate states. Instead, text changes are debounced with a 500ms delay:

```javascript
const debouncedSaveTextState = debounce(() => {
  saveState('text-modify');
}, 500);
```

Only when the user stops typing for half a second does the state get saved.

### Canvas Context Caching

The 2D rendering contexts are stored in variables rather than repeatedly calling `getContext()`. I think this is minor, but each little step in optimization adds up.

### Layer Visibility

When toggling layer visibility, the web app doesn't redraw anything. It simply sets `display: none` on the relevant canvas or text container.

## Color Scheme and Theming

Scanlateanator uses a dark theme inspired by Visual Studio Code. For anyone interested:

- **Background**: `#1e1e1e` (dark charcoal)
- **Panels**: `#2d2d30` (slightly lighter)
- **Borders**: `#3e3e42` (subtle separation)
- **Accent**: `#0e639c` (VSCode blue)
- **Text**: `#e0e0e0` (light gray)

This dark-mode-esque palette is easy on the eyes.

As for the layout, it follows a standard application structure:

- **Top bar**: Logo and primary tools
- **Left sidebar**: History and page management
- **Right sidebar**: Tool settings and text formatting
- **Center**: Large canvas workspace

## Browser Compatibility

The application is designed for modern browsers (Chrome, Firefox, Safari, Edge) and makes no attempt to support IE11. This allows use of:

- ES6+ JavaScript features
- CSS Grid and Flexbox
- The canvas 2D API with modern compositing modes
- File API and Blob URLs
- HTML5 semantic elements

### WebP Detection

Since WebP support varies, the app detects it at startup:

```javascript
const detectWebPSupportAsync = async () => {
  return new Promise((resolve) => {
    const webpData = 'data:image/webp;base64,...';
    const img = new Image();
    img.onload = () => resolve(img.width === 1 && img.height === 1);
    img.onerror = () => resolve(false);
    img.src = webpData;
  });
};
```

If WebP isn't supported, the app falls back to PNG for both internal storage and exports.

## Challenges and Solutions

### Challenge 1: Canvas Memory Usage

**Problem**: Storing full canvas state for undo consumed 30-50MB per state with high-res images.

**Solution**: Implemented region-based storage that only saves affected pixels, plus WebP compression. Reduced memory consumption heavily.

### Challenge 2: Text Rotation

**Problem**: Rotating text divs makes them hard to edit (upside-down text isn't user-friendly).

**Solution**: Store rotation as data, keep text upright during editing, only apply rotation during export rendering.

### Challenge 3: Smooth Brush Strokes

**Problem**: Mouse events fire at non-continuous intervals, causing gaps in fast strokes.

**Solution**: Interpolate between mouse positions by drawing lines between each point, creating smooth continuous strokes.

### Challenge 4: Mobile Experience

**Problem**: The web app fundamentally requires mouse and keyboard precision, and I never intended it to be for a smaller screen.

**Solution**: Instead of compromising the desktop experience or creating a poor mobile version, I decided to show a mobile landing page that explains the requirements and encourages desktop use.

### Challenge 5: Export Performance

**Problem**: Rendering text boxes with complex formatting (stroke, rotation) was slow.

**Solution**: Pre-render text to temporary canvases and cache the results, then composite them onto the final export canvas.

## Code Organization

If you'd like to go through the JavaScript file on my GitHub, it's organized into logical sections.

1. **Configuration & Constants**: Settings, feature flags, action labels
2. **DOM Helpers & Globals**: Element references, state object
3. **Utility Functions**: Debounce, bounds calculation, compression
4. **Canvas Drawing**: Brush, eraser, stroke rendering
5. **Text System**: Text box creation, manipulation, rendering
6. **History System**: Undo/redo stack management
7. **Page Management**: Multi-page state switching
8. **Zoom/Pan**: Viewport transformation
9. **Export**: Flattening and download logic
10. **Event Handlers**: Mouse and keyboard, UI interactions
11. **Initialization**: Setup and feature detection

Each section is separated by a comment block:

```javascript
/* ===================================
   SECTION NAME
   =================================== */
```

## Future Prospects

Potential improvements I've been thinking of:

- **Save/Load Projects**: Allow users to save work-in-progress and resume later
- **Selection Tool**: Select and move multiple text boxes at once
- **Shape Tools**: Add rectangles, circles for covering standard areas quickly
- **Custom Brushes**: Different brush shapes and patterns
- **Typesetting Presets**: Save and load font/formatting combinations

However, I believe that the current feature set covers most of the typical scanlation workflows without overbloating the interface.

## Inspirations

Of course, I did not come up with this idea for a web-based scanlation app by myself. I was heavily inspired by Adobe Photoshop, Pixlr, GIMP, and—most importantly—by [Scanlate.io](https://scanlate.io). Scanlate.io is a webpage that is almost one-of-a-kind in what it does. It enables users to brush out speech bubbles, erase brushes, add preset fonts, and work with multiple pages and multiple projects after logging in.

Honestly, I am in awe of the ingenuity and coding decisions made for the app. However, there were some qualms I had with the website. 

Firstly, there was a severe lack of editing features. ***Users could not***:

- **Upload custom fonts**
- **Undo/Redo**
- **Work on multiple pages without logging in**
- **Zoom/Pan**
- **Export to different file formats**

Secondly, I found the UI to be overly simple and outdated. With heavy inspirations and expectations, I set out to create my own version. 

## Conclusion

I started this project back in November, 2025, and have worked on/off for a while. It feels great to have worked out major hurdles, completing the process of debugging, to produce a functioning application. Scanlateanator is a web application that provides great functionality without requiring complex frameworks or build tools, because I have put in effort to make it optimized and specific to a unique usecase without relying on excessive external libs.

### Key Takeaways

1. Vanilla JavaScript is enough to create complex applications (though I suppose there are libraries out there that make it easier to work with)
2. Performance optimization matters, because region-based storage and compression made a great difference in memory usage
3. It's important to know your constraints: I knew I couldn't redesign the web app to be mobile-friendly, so I decided to work around that.

The complete application weighs in at just 123KB (epic number XD) total (HTML + CSS + JS files combined), loads instantly, and requires no installation. I'm always amazed when I create such complex applications using web design, spending months, and the entire codebase takes up less storage than one frame of a movie.
