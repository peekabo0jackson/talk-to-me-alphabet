# Animation-Audio Sync Bug

## Symptom
Animation visibly starts AFTER audio plays. Increasing delay pattern:
- First ~6 letters (QWERTY row) fire audio+animation in sync
- Remaining letters show increasing delay between audio start and animation start
- Happens consistently on every keypress, not just first load
- Audio timing is correct (plays instantly)
- Animation is the lagging component

## What's been ruled out
- **NOT the audio files** — trimmed with sox, verified <30ms onset for all 26
- **NOT AudioContext suspension** — fixed by Codex, audio plays fine
- **NOT code ordering** — animation class is added before audio fires in activateLetter()
- **NOT requestAnimationFrame in background decoder** — switched to setTimeout(100ms), lag persists

## Current activateLetter flow (~line 917)
```js
1. querySelector for container
2. classList.remove('active')
3. void container.offsetWidth   // forced sync reflow
4. classList.add('active')      // CSS animation should start here
5. playPhonics(letter)          // Web Audio buffer.start(0)
6. createParticles(container)   // 8 DOM elements added
```

## Hypotheses to test (need Chrome DevTools profiling)
1. **Forced reflow cost** — `void container.offsetWidth` on a 2MB page may take significant time
2. **Paint delay** — CSS animation doesn't visually start until browser paints, which happens after JS yields. Audio thread fires independently and faster.
3. **Main thread contention** — 2MB of inline base64 JS data may cause GC pressure or slow DOM operations
4. **Particle creation** — 8 createElement + appendChild calls between animation trigger and JS yield may delay the paint

## Suggested approach
Profile with Chrome DevTools Performance tab. Record a keypress and look at:
- Time between classList.add('active') and first animation frame paint
- Any long tasks between the keypress event and the paint
- Whether the forced reflow is expensive
