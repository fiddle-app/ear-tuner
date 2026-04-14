

## ## Fonts

**Inconsolata** is a monospace coding font.  It gives the app a "technical instrument" feel, like a digital tuner or signal meter.

> [!todo] Consider a non-monospace font for note names


**Nunito** is used as a secondary font that is visually compatible with Inconsolata, but has a better looking zero "0".

## Colors

The **burnt orange (#b83c08)** is bold and distinctive. It reads as warm and energetic — suits a practice tool. 
The **warm parchment (#f5efe6)** in the info overlay
The **dark forest green (#0d2a1a)** for retest mode completes a coherent 3-color family.

### Issues

- **Vignette at 55% opacity**: Too heavy on a small screen. It creates a dark tunnel effect and competes with the circle elements. Pull it back to ~28–30%.
> [!casey] I don't think that I agree
- **Status text #e8d5b0**: Warm tan on orange is low contrast. On a sunlit iPhone screen it could be hard to read. This is the most impactful usability fix — change to `rgba(255,255,255,0.92)` and give it a color shift on correct/wrong states (soft green / soft coral). Currently status text is the same color whether you got it right or wrong.

> [!todo] review this
- **Circle flash states** (correct/wrong): The `rgba` bg opacity of 0.34 is too subtle against the orange background. Both states need to be more vivid — 0.42–0.45 bg opacity and brighter borders. Currently they're easy to miss.
