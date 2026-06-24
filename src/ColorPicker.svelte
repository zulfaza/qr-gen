<script lang="ts">
  import { onMount } from 'svelte'

  // value is an 8-digit hex string: #rrggbbaa
  export let value = '#000000ff'
  export let label = 'Color'

  let open = false
  let root: HTMLElement
  let svBox: HTMLElement

  // HSVA is the source of truth while the picker is open.
  let h = 0 // 0..360
  let s = 0 // 0..1
  let v = 0 // 0..1
  let a = 1 // 0..1
  let initialized = false

  function clamp(n: number, min = 0, max = 1): number {
    return Math.min(max, Math.max(min, n))
  }

  function hsvToRgb(hh: number, ss: number, vv: number): [number, number, number] {
    const c = vv * ss
    const x = c * (1 - Math.abs(((hh / 60) % 2) - 1))
    const m = vv - c
    let r = 0
    let g = 0
    let b = 0
    if (hh < 60) [r, g, b] = [c, x, 0]
    else if (hh < 120) [r, g, b] = [x, c, 0]
    else if (hh < 180) [r, g, b] = [0, c, x]
    else if (hh < 240) [r, g, b] = [0, x, c]
    else if (hh < 300) [r, g, b] = [x, 0, c]
    else [r, g, b] = [c, 0, x]
    return [Math.round((r + m) * 255), Math.round((g + m) * 255), Math.round((b + m) * 255)]
  }

  function rgbToHsv(r: number, g: number, b: number): [number, number, number] {
    r /= 255
    g /= 255
    b /= 255
    const max = Math.max(r, g, b)
    const min = Math.min(r, g, b)
    const d = max - min
    let hh = 0
    if (d !== 0) {
      if (max === r) hh = ((g - b) / d) % 6
      else if (max === g) hh = (b - r) / d + 2
      else hh = (r - g) / d + 4
      hh *= 60
      if (hh < 0) hh += 360
    }
    const ss = max === 0 ? 0 : d / max
    return [hh, ss, max]
  }

  function toHex(n: number): string {
    return Math.round(clamp(n, 0, 255)).toString(16).padStart(2, '0')
  }

  function parseHex(hex: string): [number, number, number, number] | null {
    let raw = hex.trim().replace(/^#/, '')
    if (raw.length === 3) raw = raw.replace(/./g, (c) => c + c)
    if (raw.length === 6) raw += 'ff'
    if (raw.length !== 8 || /[^0-9a-fA-F]/.test(raw)) return null
    const r = parseInt(raw.slice(0, 2), 16)
    const g = parseInt(raw.slice(2, 4), 16)
    const b = parseInt(raw.slice(4, 6), 16)
    const al = parseInt(raw.slice(6, 8), 16) / 255
    return [r, g, b, al]
  }

  // Derived display values.
  $: rgb = hsvToRgb(h, s, v)
  $: hex6 = `#${toHex(rgb[0])}${toHex(rgb[1])}${toHex(rgb[2])}`
  $: alphaPercent = Math.round(a * 100)
  $: hueColor = `hsl(${h}, 100%, 50%)`
  $: solidColor = `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`
  $: swatchColor = `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, ${a})`

  // Push HSVA back up into `value` once we've taken control.
  $: if (initialized) {
    value = `${hex6}${toHex(a * 255)}`
  }

  function syncFromValue(hex: string): void {
    const parsed = parseHex(hex)
    if (!parsed) return
    const [r, g, b, al] = parsed
    const [nh, ns, nv] = rgbToHsv(r, g, b)
    // Preserve hue when the color is grayscale (saturation 0).
    if (ns !== 0) h = nh
    s = ns
    v = nv
    a = al
  }

  onMount(() => {
    syncFromValue(value)
    initialized = true
  })

  function pointFromEvent(event: PointerEvent, el: HTMLElement): [number, number] {
    const rect = el.getBoundingClientRect()
    return [clamp((event.clientX - rect.left) / rect.width), clamp((event.clientY - rect.top) / rect.height)]
  }

  function startDrag(
    event: PointerEvent,
    el: HTMLElement,
    onMove: (x: number, y: number) => void,
  ): void {
    event.preventDefault()
    const [x, y] = pointFromEvent(event, el)
    onMove(x, y)
    const move = (e: PointerEvent) => {
      const [nx, ny] = pointFromEvent(e, el)
      onMove(nx, ny)
    }
    const up = () => {
      window.removeEventListener('pointermove', move)
      window.removeEventListener('pointerup', up)
    }
    window.addEventListener('pointermove', move)
    window.addEventListener('pointerup', up)
  }

  function onSvDown(event: PointerEvent): void {
    startDrag(event, svBox, (x, y) => {
      s = x
      v = 1 - y
    })
  }

  function onHueInput(event: Event): void {
    h = Number((event.target as HTMLInputElement).value)
  }

  function onAlphaInput(event: Event): void {
    a = Number((event.target as HTMLInputElement).value) / 100
  }

  function onHexInput(event: Event): void {
    const parsed = parseHex((event.target as HTMLInputElement).value)
    if (!parsed) return
    const [r, g, b, al] = parsed
    const [nh, ns, nv] = rgbToHsv(r, g, b)
    if (ns !== 0) h = nh
    s = ns
    v = nv
    a = al
  }

  function onAlphaText(event: Event): void {
    const n = Number((event.target as HTMLInputElement).value)
    if (Number.isNaN(n)) return
    a = clamp(n, 0, 100) / 100
  }

  const hasEyeDropper = typeof window !== 'undefined' && 'EyeDropper' in window

  async function pickFromScreen(): Promise<void> {
    try {
      // @ts-expect-error EyeDropper is not in the TS DOM lib yet.
      const result = await new window.EyeDropper().open()
      syncFromValue(result.sRGBHex)
    } catch {
      // user cancelled
    }
  }

  function toggle(): void {
    open = !open
    if (open) syncFromValue(value)
  }

  function onWindowPointerDown(event: PointerEvent): void {
    if (open && root && !root.contains(event.target as Node)) open = false
  }

  function onKeydown(event: KeyboardEvent): void {
    if (event.key === 'Escape') open = false
  }
</script>

<svelte:window on:pointerdown={onWindowPointerDown} on:keydown={onKeydown} />

<div class="picker" bind:this={root}>
  <button type="button" class="picker-trigger" on:click={toggle} aria-expanded={open}>
    <span class="picker-swatch" style="--c: {swatchColor}"></span>
    <span class="picker-trigger-text">{hex6}{alphaPercent < 100 ? ` · ${alphaPercent}%` : ''}</span>
  </button>

  {#if open}
    <div class="picker-panel" role="dialog" aria-label={label}>
      <div
        class="picker-sv"
        bind:this={svBox}
        on:pointerdown={onSvDown}
        style="--hue: {hueColor}"
      >
        <div class="picker-sv-handle" style="left: {s * 100}%; top: {(1 - v) * 100}%; --c: {solidColor}"></div>
      </div>

      <input
        class="picker-slider picker-hue"
        type="range"
        min="0"
        max="360"
        step="1"
        value={h}
        on:input={onHueInput}
        aria-label="Hue"
      />

      <input
        class="picker-slider picker-alpha"
        type="range"
        min="0"
        max="100"
        step="1"
        value={alphaPercent}
        on:input={onAlphaInput}
        aria-label="Opacity"
        style="--solid: {solidColor}"
      />

      <div class="picker-fields">
        {#if hasEyeDropper}
          <button type="button" class="picker-eyedropper" on:click={pickFromScreen} aria-label="Pick color from screen">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="m2 22 1-1h3l9-9" />
              <path d="M3 21v-3l9-9" />
              <path d="m15 6 3.5-3.5a2.12 2.12 0 0 1 3 3L18 9l.5.5a1.4 1.4 0 0 1 0 2 1.4 1.4 0 0 1-2 0l-4-4a1.4 1.4 0 0 1 0-2 1.4 1.4 0 0 1 2 0Z" />
            </svg>
          </button>
        {/if}
        <span class="picker-format">HEX</span>
        <input
          class="picker-hex"
          type="text"
          value={hex6}
          on:change={onHexInput}
          spellcheck="false"
          aria-label="Hex value"
        />
        <span class="picker-alpha-field">
          <input
            class="picker-alpha-text"
            type="number"
            min="0"
            max="100"
            value={alphaPercent}
            on:change={onAlphaText}
            aria-label="Opacity percent"
          />
          <span>%</span>
        </span>
      </div>
    </div>
  {/if}
</div>

<style>
  .picker {
    position: relative;
  }

  .picker-trigger {
    width: 100%;
    min-height: 44px;
    justify-content: flex-start;
    gap: 10px;
    padding: 0 12px;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: #ffffff;
    color: var(--text);
  }

  .picker-trigger:hover:not(:disabled) {
    background: #ffffff;
    border-color: var(--border-strong);
  }

  .picker-swatch {
    width: 26px;
    height: 26px;
    border-radius: 6px;
    border: 1px solid rgba(15, 23, 42, 0.12);
    background-image: linear-gradient(var(--c), var(--c)),
      conic-gradient(#cbd5e1 0 25%, #fff 0 50%, #cbd5e1 0 75%, #fff 0);
    background-size:
      100% 100%,
      12px 12px;
  }

  .picker-trigger-text {
    font-weight: 400;
    font-variant-numeric: tabular-nums;
    text-transform: uppercase;
    color: var(--muted);
  }

  .picker-panel {
    position: absolute;
    z-index: 30;
    top: calc(100% + 8px);
    left: 0;
    width: 280px;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 14px;
    border: 1px solid var(--border);
    border-radius: 14px;
    background: #ffffff;
    box-shadow: 0 16px 40px rgba(15, 23, 42, 0.18);
  }

  .picker-sv {
    position: relative;
    width: 100%;
    height: 160px;
    border-radius: 10px;
    cursor: crosshair;
    touch-action: none;
    background:
      linear-gradient(to top, #000, transparent),
      linear-gradient(to right, #fff, transparent),
      var(--hue);
  }

  .picker-sv-handle {
    position: absolute;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    border: 2px solid #fff;
    background: var(--c);
    box-shadow: 0 0 0 1px rgba(15, 23, 42, 0.35);
    pointer-events: none;
  }

  .picker-slider {
    -webkit-appearance: none;
    appearance: none;
    width: 100%;
    height: 14px;
    border-radius: 999px;
    margin: 0;
    cursor: pointer;
    touch-action: none;
    border: 1px solid rgba(15, 23, 42, 0.08);
  }

  .picker-hue {
    background: linear-gradient(
      to right,
      #ff0000,
      #ffff00,
      #00ff00,
      #00ffff,
      #0000ff,
      #ff00ff,
      #ff0000
    );
  }

  .picker-alpha {
    background-image: linear-gradient(to right, transparent, var(--solid)),
      conic-gradient(#cbd5e1 0 25%, #fff 0 50%, #cbd5e1 0 75%, #fff 0);
    background-size:
      100% 100%,
      12px 12px;
  }

  .picker-slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    border: 1px solid rgba(15, 23, 42, 0.25);
    box-shadow: 0 1px 3px rgba(15, 23, 42, 0.3);
    cursor: pointer;
  }

  .picker-slider::-moz-range-thumb {
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    border: 1px solid rgba(15, 23, 42, 0.25);
    box-shadow: 0 1px 3px rgba(15, 23, 42, 0.3);
    cursor: pointer;
  }

  .picker-fields {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .picker-eyedropper {
    min-height: 38px;
    width: 38px;
    flex: 0 0 auto;
    padding: 0;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: #ffffff;
    color: var(--muted);
  }

  .picker-eyedropper:hover:not(:disabled) {
    background: var(--panel-soft);
    border-color: var(--border-strong);
    color: var(--text);
  }

  .picker-format {
    font-size: 13px;
    font-weight: 700;
    color: var(--muted);
  }

  .picker-hex {
    flex: 1 1 auto;
    min-width: 0;
    width: 100%;
    min-height: 38px;
    padding: 0 10px;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: var(--panel-soft);
    color: var(--text);
    font: inherit;
    font-size: 14px;
    font-variant-numeric: tabular-nums;
    text-transform: uppercase;
  }

  .picker-alpha-field {
    display: flex;
    align-items: center;
    gap: 2px;
    flex: 0 0 auto;
    min-height: 38px;
    padding: 0 8px;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: var(--panel-soft);
    color: var(--muted);
    font-size: 14px;
  }

  .picker-alpha-text {
    width: 36px;
    border: none;
    background: transparent;
    color: var(--text);
    font: inherit;
    font-size: 14px;
    font-variant-numeric: tabular-nums;
    text-align: right;
    -moz-appearance: textfield;
  }

  .picker-alpha-text::-webkit-outer-spin-button,
  .picker-alpha-text::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }

  .picker-hex:focus-visible,
  .picker-alpha-text:focus-visible,
  .picker-eyedropper:focus-visible,
  .picker-trigger:focus-visible {
    outline: 3px solid var(--focus);
    outline-offset: 2px;
  }
</style>
