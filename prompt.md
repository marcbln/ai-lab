Act as an expert WebGL developer. Create a complete, self-contained, and highly polished single-file WebGL demonstration using Three.js (imported via CDN). 

The concept is "Alien Invasion: City Destroyer." It features multiple switchable cities being actively destroyed by alien motherships, with interactive controls and dynamic visual effects.

### Technical Stack & Setup
- Use a single HTML file with embedded CSS and ES6 JavaScript.
- Include CDN links for:
  - Three.js (latest stable r128 or higher)
  - OrbitControls
  - GSAP (for smooth animations and transitions)
  - lil-gui (for user controls)
- The app must handle window resizing seamlessly and run at a smooth 60fps using requestAnimationFrame.

### Visual Style & Theme
- **Aesthetics:** Cyberpunk/Futuristic. Use dark skies, heavy fog (THREE.FogExp2) to create scale, and highly emissive glow colors (neon blues, purples, and destructive oranges/reds).
- **Procedural Cities:** Since we cannot load external 3D models, generate cities procedurally. Implement a `CityGenerator` class that constructs 3 different switchable city profiles:
  1. **New York (Manhattan):** High-density grid of towering rectangular skyscrapers. Add a central "Empire State" style tower.
  2. **Paris:** Low-rise, uniform blocks with a prominent, procedurally-built Eiffel Tower in the center (made of stacked wireframe pyramids/cylinders).
  3. **Berlin:** A mix of blocky industrial/brutalist buildings and a distinct central TV Tower (Fernsehturm) with a sphere and spire.
- **Building Details:** 
  - Do not use plain boxes. Create a custom shader or a canvas-generated texture to map glowing "window grids" onto the building faces.
  - Give buildings varied heights, widths, and neon rooftop antennas.

### The Alien Invaders & Destruction
- **The Mothership:** Position a large, detailed alien mothership (a glowing, rotating disc or ring structure) high above the center of the city. Use glowing emissive materials.
- **The Attack (Laser Beams):** 
  - Periodically, or when the user clicks, the mothership fires a massive, glowing laser beam (a cylinder with a custom shader or additive blending) down onto a building.
  - Implement a glowing impact ring (expanding circle) on the ground plane where the laser hits.
- **Destruction Logic:**
  - When a building is hit by a laser, it should "crumble." Animate its scale.y down to 0.1, turn its color to a charred black/glowing red, and trigger a dramatic particle explosion at its base.
  - Use a particle system (Points) for fire and smoke. Particles should rise, fade, and disperse using physics-like velocity.

### Interactivity & User Interface
- **Orbit Camera:** Allow the user to rotate, zoom, and pan around the city.
- **Interactive Targeting:** Implement a Raycaster. When the user clicks on any building:
  - The mothership rotates towards the clicked target.
  - A laser fires from the mothership to the clicked building.
  - The building crumbles, accompanied by a localized particle explosion.
- **UI Control Panel (lil-gui or styled HTML overlay in the top-right):**
  - **City Selector dropdown:** (New York, Paris, Berlin) -> Smoothly transition between cities by dissolving the old one (shrinking/fading out) and spawning the new one.
  - **Invasion Intensity slider:** Adjusts how frequently the mothership automatically targets and destroys buildings.
  - **"Reset City" button:** Rebuilds all destroyed structures.
  - **"Armageddon" button:** Fires lasers rapidly at multiple random buildings at once.

### Performance Optimizations
- Use `THREE.InstancedMesh` for the generic buildings if possible to keep draw calls low, or merge building geometries where appropriate.
- Limit the max particle count for explosions to keep the framerate high.
- Clean up geometries, materials, and textures from memory when switching between cities to prevent memory leaks.

Please write the complete, well-commented code, ensuring no placeholders or "TODOs" are left behind. Make sure the visual effects (glows, particle motion, and transitions) look fluid and professional.

