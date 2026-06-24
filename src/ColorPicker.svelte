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
  let lastSyncedValue = value

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
  $: nextValue = `${hex6}${toHex(a * 255)}`

  // Keep internal HSVA and externally controlled values in lockstep.
  $: if (initialized && value !== lastSyncedValue && value !== nextValue) {
    syncFromValue(value)
    lastSyncedValue = value
  }

  $: if (initialized && value !== nextValue) {
    lastSyncedValue = nextValue
    value = nextValue
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
    lastSyncedValue = value
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

  function getInputElement(target: EventTarget | null): HTMLInputElement | undefined {
    return target instanceof HTMLInputElement ? target : undefined
  }

  function onHueInput(event: Event): void {
    const input = getInputElement(event.target)
    if (!input) return
    h = Number(input.value)
  }

  function onAlphaInput(event: Event): void {
    const input = getInputElement(event.target)
    if (!input) return
    a = Number(input.value) / 100
  }

  function onHexInput(event: Event): void {
    const input = getInputElement(event.target)
    if (!input) return
    const parsed = parseHex(input.value)
    if (!parsed) return
    const [r, g, b, al] = parsed
    const [nh, ns, nv] = rgbToHsv(r, g, b)
    if (ns !== 0) h = nh
    s = ns
    v = nv
    a = al
  }

  function onAlphaText(event: Event): void {
    const input = getInputElement(event.target)
    if (!input) return
    const n = Number(input.value)
    if (Number.isNaN(n)) return
    a = clamp(n, 0, 100) / 100
  }

  const hasEyeDropper = typeof window !== 'undefined' && window.EyeDropper !== undefined

  // Shared range-slider styling: native appearance reset plus custom thumbs
  // for both WebKit and Firefox. Reused by the hue and alpha sliders.
  const sliderClass =
    'm-0 h-[14px] w-full cursor-pointer touch-none appearance-none border border-[rgba(15,23,42,0.08)] ' +
    '[&::-webkit-slider-thumb]:h-[18px] [&::-webkit-slider-thumb]:w-[18px] [&::-webkit-slider-thumb]:appearance-none [&::-webkit-slider-thumb]:cursor-pointer [&::-webkit-slider-thumb]:rounded-full [&::-webkit-slider-thumb]:border [&::-webkit-slider-thumb]:border-[rgba(15,23,42,0.25)] [&::-webkit-slider-thumb]:bg-white [&::-webkit-slider-thumb]:shadow-[0_1px_3px_rgba(15,23,42,0.3)] ' +
    '[&::-moz-range-thumb]:h-[18px] [&::-moz-range-thumb]:w-[18px] [&::-moz-range-thumb]:cursor-pointer [&::-moz-range-thumb]:rounded-full [&::-moz-range-thumb]:border [&::-moz-range-thumb]:border-[rgba(15,23,42,0.25)] [&::-moz-range-thumb]:bg-white [&::-moz-range-thumb]:shadow-[0_1px_3px_rgba(15,23,42,0.3)]'

  async function pickFromScreen(): Promise<void> {
    try {
      if (!window.EyeDropper) return
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
    if (open && root && event.target instanceof Node && !root.contains(event.target)) open = false
  }

  function onKeydown(event: KeyboardEvent): void {
    if (event.key === 'Escape') open = false
  }
</script>

<svelte:window on:pointerdown={onWindowPointerDown} on:keydown={onKeydown} />

<div class="relative" bind:this={root}>
  <button
    type="button"
    class="focus-ring inline-flex min-h-16 w-full cursor-pointer items-center justify-start gap-4 border border-text bg-panel px-5 text-text hover:border-accent"
    on:click={toggle}
    aria-expanded={open}
  >
    <span
      class="h-8 w-8 border border-border-strong [background-image:linear-gradient(var(--c),var(--c)),conic-gradient(#cbd5e1_0_25%,#fff_0_50%,#cbd5e1_0_75%,#fff_0)] [background-size:100%_100%,12px_12px]"
      style="--c: {swatchColor}"
    ></span>
    <span class="text-[18px] font-bold uppercase tabular-nums text-text">{hex6}{alphaPercent < 100 ? ` · ${alphaPercent}%` : ''}</span>
  </button>

  {#if open}
    <div
      class="absolute left-0 top-[calc(100%+8px)] z-30 flex w-[280px] flex-col gap-[14px] border border-text bg-panel p-4 shadow-[0_8px_8px_rgba(16,20,23,0.12)]"
      role="dialog"
      aria-label={label}
    >
      <div
        class="relative h-40 w-full cursor-crosshair touch-none [background:linear-gradient(to_top,#000,transparent),linear-gradient(to_right,#fff,transparent),var(--hue)]"
        bind:this={svBox}
        on:pointerdown={onSvDown}
        role="slider"
        tabindex="0"
        aria-label="{label} saturation and brightness"
        aria-valuemin="0"
        aria-valuemax="100"
        aria-valuenow={Math.round(s * 100)}
        aria-valuetext="{Math.round(s * 100)} percent saturation, {Math.round(v * 100)} percent brightness"
        style="--hue: {hueColor}"
      >
        <div
          class="pointer-events-none absolute h-[18px] w-[18px] -translate-x-1/2 -translate-y-1/2 rounded-full border-2 border-white shadow-[0_0_0_1px_rgba(15,23,42,0.35)]"
          style="left: {s * 100}%; top: {(1 - v) * 100}%; background: {solidColor}"
        ></div>
      </div>

      <input
        class="{sliderClass} [background:linear-gradient(to_right,#ff0000,#ffff00,#00ff00,#00ffff,#0000ff,#ff00ff,#ff0000)]"
        type="range"
        min="0"
        max="360"
        step="1"
        value={h}
        on:input={onHueInput}
        aria-label="Hue"
      />

      <input
        class="{sliderClass} [background-image:linear-gradient(to_right,transparent,var(--solid)),conic-gradient(#cbd5e1_0_25%,#fff_0_50%,#cbd5e1_0_75%,#fff_0)] [background-size:100%_100%,12px_12px]"
        type="range"
        min="0"
        max="100"
        step="1"
        value={alphaPercent}
        on:input={onAlphaInput}
        aria-label="Opacity"
        style="--solid: {solidColor}"
      />

      <div class="flex items-center gap-2">
        {#if hasEyeDropper}
          <button
            type="button"
            class="focus-ring inline-flex h-[38px] w-[38px] flex-none cursor-pointer items-center justify-center border border-border bg-panel p-0 text-muted hover:border-text hover:text-text"
            on:click={pickFromScreen}
            aria-label="Pick color from screen"
          >
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="m2 22 1-1h3l9-9" />
              <path d="M3 21v-3l9-9" />
              <path d="m15 6 3.5-3.5a2.12 2.12 0 0 1 3 3L18 9l.5.5a1.4 1.4 0 0 1 0 2 1.4 1.4 0 0 1-2 0l-4-4a1.4 1.4 0 0 1 0-2 1.4 1.4 0 0 1 2 0Z" />
            </svg>
          </button>
        {/if}
        <span class="text-[13px] font-bold text-muted">HEX</span>
        <input
          class="focus-ring min-h-[38px] w-full min-w-0 flex-1 border border-border bg-panel-soft px-2.5 text-sm uppercase tabular-nums text-text"
          type="text"
          value={hex6}
          on:change={onHexInput}
          spellcheck="false"
          aria-label="Hex value"
        />
        <span class="flex min-h-[38px] flex-none items-center gap-0.5 border border-border bg-panel-soft px-2 text-sm text-muted">
          <input
            class="focus-ring w-9 border-0 bg-transparent text-right text-sm tabular-nums text-text [-moz-appearance:textfield] [&::-webkit-inner-spin-button]:m-0 [&::-webkit-inner-spin-button]:appearance-none [&::-webkit-outer-spin-button]:m-0 [&::-webkit-outer-spin-button]:appearance-none"
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
