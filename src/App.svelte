<script lang="ts">
  import { onMount } from 'svelte'
  import { toDataURL, toString } from 'qrcode'
  import type { QRCodeToDataURLOptions, QRCodeToStringOptions } from 'qrcode'
  import ColorPicker from './ColorPicker.svelte'
  import QrReader from './QrReader.svelte'

  type AppTab = 'generate' | 'read'
  type DownloadFormat = 'png' | 'svg'
  type UrlState = {
    input: string
    darkColor: string
    lightColor: string
    logoColor: string
  }

  type QrState =
    | { kind: 'idle' }
    | { kind: 'generating'; text: string }
    | { kind: 'ready'; text: string; pngDataUrl: string; svgMarkup: string }
    | { kind: 'failed'; message: string }

  // Fraction of the QR width occupied by the logo. Kept conservative so the
  // logo never obscures enough modules to break scanning.
  const LOGO_SCALE = 0.22
  const DEFAULT_URL_STATE: UrlState = {
    input: 'hello there',
    darkColor: '#14492fff',
    lightColor: '#ffffffff',
    logoColor: '#ffffffff',
  }

  const initialUrlState = readInitialUrlState()

  let activeTab: AppTab = 'generate'
  let input = initialUrlState.input
  let darkColor = initialUrlState.darkColor
  let lightColor = initialUrlState.lightColor
  let logoColor = initialUrlState.logoColor
  let logoDataUrl = ''
  let logoInput: HTMLInputElement | undefined
  let qrState: QrState = { kind: 'idle' }
  let busy = false
  let requestId = 0
  let urlSyncReady = false

  $: void renderQr(input, darkColor, lightColor, logoColor, logoDataUrl)
  $: if (urlSyncReady && activeTab === 'generate') {
    writeUrlState({ input, darkColor, lightColor, logoColor })
  }
  $: encodedLength = getEncodedLength(qrState, input)

  onMount(() => {
    urlSyncReady = true

    const onPopState = () => {
      const state = readUrlState(window.location.search)
      input = state.input
      darkColor = state.darkColor
      lightColor = state.lightColor
      logoColor = state.logoColor
    }

    window.addEventListener('popstate', onPopState)
    return () => window.removeEventListener('popstate', onPopState)
  })

  function readInitialUrlState(): UrlState {
    if (typeof window === 'undefined') {
      return DEFAULT_URL_STATE
    }

    return readUrlState(window.location.search)
  }

  function readUrlState(search: string): UrlState {
    const params = new URLSearchParams(search)
    return {
      input: params.get('text') ?? DEFAULT_URL_STATE.input,
      darkColor: parseUrlColor(params.get('dark'), DEFAULT_URL_STATE.darkColor),
      lightColor: parseUrlColor(params.get('light'), DEFAULT_URL_STATE.lightColor),
      logoColor: parseUrlColor(params.get('panel'), DEFAULT_URL_STATE.logoColor),
    }
  }

  function parseUrlColor(value: string | null, fallback: string): string {
    if (!value) {
      return fallback
    }

    const normalized = value.startsWith('#') ? value : `#${value}`
    return /^#[0-9a-fA-F]{8}$/.test(normalized) ? normalized.toLowerCase() : fallback
  }

  function writeUrlState(state: UrlState): void {
    const params = new URLSearchParams()
    params.set('text', state.input)
    params.set('dark', stripColorHash(state.darkColor))
    params.set('light', stripColorHash(state.lightColor))
    params.set('panel', stripColorHash(state.logoColor))

    const nextUrl = `${window.location.pathname}?${params.toString()}${window.location.hash}`
    const currentUrl = `${window.location.pathname}${window.location.search}${window.location.hash}`
    if (nextUrl !== currentUrl) {
      window.history.replaceState(null, '', nextUrl)
    }
  }

  function stripColorHash(color: string): string {
    return color.startsWith('#') ? color.slice(1) : color
  }

  function buildPngOptions(dark: string, light: string, hasLogo: boolean): QRCodeToDataURLOptions {
    return {
      type: 'image/png',
      errorCorrectionLevel: hasLogo ? 'H' : 'M',
      margin: 2,
      width: 960,
      color: { dark, light },
    }
  }

  function buildSvgOptions(dark: string, light: string, hasLogo: boolean): QRCodeToStringOptions {
    return {
      type: 'svg',
      errorCorrectionLevel: hasLogo ? 'H' : 'M',
      margin: 2,
      width: 512,
      color: { dark, light },
    }
  }

  function loadImage(src: string): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
      const image = new Image()
      image.onload = () => resolve(image)
      image.onerror = () => reject(new Error('Could not load the logo image.'))
      image.src = src
    })
  }

  // Draw the logo centered on the QR PNG, sitting on a rounded light panel so
  // it stays legible against the modules.
  async function compositeLogoPng(qrDataUrl: string, logo: string, panelColor: string): Promise<string> {
    const [qrImage, logoImage] = await Promise.all([loadImage(qrDataUrl), loadImage(logo)])
    const size = qrImage.width
    const canvas = document.createElement('canvas')
    canvas.width = size
    canvas.height = size
    const ctx = canvas.getContext('2d')
    if (!ctx) {
      throw new Error('Canvas is not available for compositing the logo.')
    }

    ctx.drawImage(qrImage, 0, 0, size, size)

    const box = size * LOGO_SCALE
    const pad = box * 0.16
    const panel = box + pad * 2
    const x = (size - panel) / 2
    const y = (size - panel) / 2
    const radius = panel * 0.22

    ctx.fillStyle = panelColor
    roundRect(ctx, x, y, panel, panel, radius)
    ctx.fill()

    // Preserve the logo aspect ratio inside the reserved box.
    const ratio = logoImage.width / logoImage.height
    const drawW = ratio >= 1 ? box : box * ratio
    const drawH = ratio >= 1 ? box / ratio : box
    ctx.drawImage(logoImage, (size - drawW) / 2, (size - drawH) / 2, drawW, drawH)

    return canvas.toDataURL('image/png')
  }

  function roundRect(
    ctx: CanvasRenderingContext2D,
    x: number,
    y: number,
    w: number,
    h: number,
    r: number,
  ): void {
    ctx.beginPath()
    ctx.moveTo(x + r, y)
    ctx.arcTo(x + w, y, x + w, y + h, r)
    ctx.arcTo(x + w, y + h, x, y + h, r)
    ctx.arcTo(x, y + h, x, y, r)
    ctx.arcTo(x, y, x + w, y, r)
    ctx.closePath()
  }

  // Inject the logo into the QR SVG: a rounded light panel plus an <image>,
  // both sized off the SVG's own viewBox so they scale with the markup.
  function compositeLogoSvg(svgMarkup: string, logo: string, panelColor: string): string {
    const viewBox = svgMarkup.match(/viewBox="0 0 (\d+(?:\.\d+)?) /)
    if (!viewBox) {
      return svgMarkup
    }
    const size = parseFloat(viewBox[1])
    const box = size * LOGO_SCALE
    const pad = box * 0.16
    const panel = box + pad * 2
    const px = (size - panel) / 2
    const py = (size - panel) / 2
    const radius = panel * 0.22
    const lx = (size - box) / 2
    const ly = (size - box) / 2

    const overlay =
      `<rect x="${px}" y="${py}" width="${panel}" height="${panel}" rx="${radius}" ry="${radius}" fill="${panelColor}"/>` +
      `<image x="${lx}" y="${ly}" width="${box}" height="${box}" preserveAspectRatio="xMidYMid meet" href="${logo}"/>`

    return svgMarkup.replace('</svg>', `${overlay}</svg>`)
  }

  function onLogoChange(event: Event): void {
    const file = getLogoFile(event.target)
    if (!file) {
      return
    }
    const reader = new FileReader()
    reader.onload = () => {
      logoDataUrl = typeof reader.result === 'string' ? reader.result : ''
    }
    reader.readAsDataURL(file)
  }

  function getLogoFile(target: EventTarget | null): File | undefined {
    if (!(target instanceof HTMLInputElement)) {
      return undefined
    }

    return target.files?.[0]
  }

  function openLogoPicker(): void {
    logoInput?.click()
  }

  function clearLogo(): void {
    logoDataUrl = ''
    if (logoInput) {
      logoInput.value = ''
    }
  }

  async function renderQr(
    rawText: string,
    dark: string,
    light: string,
    panel: string,
    logo: string,
  ): Promise<void> {
    const text = rawText.trim()
    const currentRequestId = requestId + 1
    requestId = currentRequestId

    if (text.length === 0) {
      busy = false
      qrState = { kind: 'idle' }
      return
    }

    busy = true
    // Keep the previous QR on screen while regenerating to avoid flicker;
    // only show the placeholder when there is nothing to show yet.
    if (qrState.kind !== 'ready') {
      qrState = { kind: 'generating', text }
    }

    try {
      const hasLogo = logo.length > 0
      let [pngDataUrl, svgMarkup] = await Promise.all([
        toDataURL(text, buildPngOptions(dark, light, hasLogo)),
        toString(text, buildSvgOptions(dark, light, hasLogo)),
      ])

      if (hasLogo) {
        ;[pngDataUrl, svgMarkup] = await Promise.all([
          compositeLogoPng(pngDataUrl, logo, panel),
          Promise.resolve(compositeLogoSvg(svgMarkup, logo, panel)),
        ])
      }

      if (currentRequestId !== requestId) {
        return
      }

      busy = false
      qrState = { kind: 'ready', text, pngDataUrl, svgMarkup }
    } catch (error: unknown) {
      if (currentRequestId !== requestId) {
        return
      }

      busy = false
      qrState = {
        kind: 'failed',
        message: error instanceof Error ? error.message : 'QR generation failed.',
      }
    }
  }

  function buildFilename(text: string, format: DownloadFormat): string {
    const slug = text
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/^-|-$/g, '')
      .slice(0, 48)

    const name = slug.length > 0 ? slug : 'qr-code'
    return `${name}.${format}`
  }

  function buildSvgDataUrl(svgMarkup: string): string {
    return `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svgMarkup)}`
  }

  function getEncodedLength(state: QrState, rawText: string): number {
    switch (state.kind) {
      case 'idle':
      case 'failed':
        return rawText.trim().length
      case 'generating':
      case 'ready':
        return state.text.length
    }
  }

  function editFromScan(text: string): void {
    input = text
    activeTab = 'generate'
  }
</script>

<main class="min-h-svh bg-panel text-text">
  <section class="mx-auto flex min-h-svh w-full max-w-[1440px] flex-col border border-text" aria-labelledby="page-title">
    <header
      class="grid min-h-[220px] items-end gap-8 border-b border-text px-6 py-8 min-[760px]:grid-cols-[minmax(0,1fr)_minmax(260px,400px)] min-[760px]:px-12 min-[760px]:py-8 min-[1120px]:min-h-[190px] min-[1120px]:px-[90px] min-[1120px]:py-8"
    >
      <div>
        <h1
          id="page-title"
          class="text-balance text-[60px] font-[900] leading-[0.92] tracking-[-0.04em] min-[560px]:text-[78px] min-[760px]:text-[68px]"
        >
          QR<br />{activeTab === 'generate' ? 'Generator' : 'Reader'}
        </h1>
        <nav class="mt-8 flex flex-wrap gap-2" aria-label="App sections">
          <button
            type="button"
            class="focus-ring cursor-pointer border px-5 py-3 text-[14px] font-[900] uppercase tracking-[0.14em] transition-colors duration-[180ms] {activeTab === 'generate'
              ? 'border-text bg-accent text-panel'
              : 'border-border-strong bg-panel text-muted hover:border-text hover:text-text'}"
            aria-current={activeTab === 'generate' ? 'page' : undefined}
            on:click={() => (activeTab = 'generate')}
          >
            Generate
          </button>
          <button
            type="button"
            class="focus-ring cursor-pointer border px-5 py-3 text-[14px] font-[900] uppercase tracking-[0.14em] transition-colors duration-[180ms] {activeTab === 'read'
              ? 'border-text bg-accent text-panel'
              : 'border-border-strong bg-panel text-muted hover:border-text hover:text-text'}"
            aria-current={activeTab === 'read' ? 'page' : undefined}
            on:click={() => (activeTab = 'read')}
          >
            Read
          </button>
        </nav>
      </div>
      <p class="max-w-[360px] self-end text-pretty text-[18px] font-semibold leading-[1.35] text-muted min-[760px]:justify-self-end min-[760px]:text-right">
        {activeTab === 'generate'
          ? 'Type once. Export a clean QR as PNG or SVG.'
          : 'Scan with your camera or upload an image to decode a QR code.'}
      </p>
    </header>

    {#if activeTab === 'read'}
      <QrReader onEditQr={editFromScan} />
    {:else}
    <div class="grid flex-1 min-[900px]:grid-cols-[minmax(0,1fr)_39%]">
      <div class="border-b border-text px-6 py-10 min-[760px]:px-12 min-[900px]:border-b-0 min-[900px]:border-r min-[1120px]:px-[90px] min-[1120px]:py-10">
        <div class="flex items-baseline justify-between gap-4">
          <label for="qr-input" class="text-[15px] font-[900] uppercase tracking-[0.28em]">Content</label>
          <span class="text-[15px] font-bold uppercase tracking-[0.16em] text-subtle">{encodedLength} characters</span>
        </div>
        <textarea
          id="qr-input"
          bind:value={input}
          rows="7"
          spellcheck="false"
          placeholder="Paste text or URL"
          aria-describedby="qr-input-hint"
          class="focus-ring mt-4 h-[190px] w-full resize-y border border-text bg-panel p-6 text-[24px] font-semibold leading-[1.3] text-text transition-colors duration-[180ms] placeholder:text-muted hover:border-accent focus-visible:border-accent min-[820px]:h-[180px]"
        ></textarea>
        <p id="qr-input-hint" class="mt-4 text-[17px] font-semibold text-subtle">
          Leading and trailing whitespace is trimmed before encoding.
        </p>

        <fieldset class="m-0 mt-10 border-0 p-0">
          <div class="flex items-center gap-8">
            <legend class="text-[15px] font-[900] uppercase tracking-[0.28em]">Colors</legend>
            <div class="h-px flex-1 bg-border"></div>
          </div>
          <div class="mt-5 flex flex-wrap gap-8">
            <div class="flex flex-1 basis-56 flex-col gap-2 text-[18px] font-semibold text-muted">
              <span>QR color</span>
              <ColorPicker bind:value={darkColor} label="QR color" />
            </div>
            <div class="flex flex-1 basis-56 flex-col gap-2 text-[18px] font-semibold text-muted">
              <span>Background</span>
              <ColorPicker bind:value={lightColor} label="Background color" />
            </div>
          </div>
        </fieldset>

        <fieldset class="m-0 mt-10 border-0 p-0">
          <div class="flex items-center gap-8">
            <legend class="text-[15px] font-[900] uppercase tracking-[0.28em]">Logo</legend>
            <div class="h-px flex-1 bg-border"></div>
          </div>
          <div class="mt-6 flex items-center gap-7">
            {#if logoDataUrl}
              <img
                class="h-20 w-20 flex-none border border-border bg-panel object-contain"
                src={logoDataUrl}
                alt="Selected logo preview"
              />
            {:else}
              <button
                type="button"
                class="focus-ring grid h-20 w-20 flex-none cursor-pointer place-items-center border border-dashed border-border-strong bg-panel text-[38px] font-light leading-none text-subtle hover:border-accent hover:text-accent"
                on:click={openLogoPicker}
                aria-label="Upload logo"
              >
                +
              </button>
            {/if}
            {#if logoDataUrl}
              <div class="flex min-w-0 flex-1 flex-col gap-3 text-[18px] font-semibold leading-[1.45] text-muted">
                <div class="flex max-w-sm flex-col gap-2">
                  <span>Logo background</span>
                  <ColorPicker bind:value={logoColor} label="Logo background color" />
                </div>
                <div class="flex flex-wrap gap-2">
                  <button
                    type="button"
                    class="focus-ring cursor-pointer border border-text bg-panel px-4 py-2 text-sm font-[900] uppercase tracking-[0.12em] text-text hover:bg-accent hover:text-panel"
                    on:click={openLogoPicker}
                  >
                    Replace
                  </button>
                  <button
                    type="button"
                    class="focus-ring cursor-pointer border border-border-strong bg-panel px-4 py-2 text-sm font-[900] uppercase tracking-[0.12em] text-muted hover:border-text hover:text-text"
                    on:click={clearLogo}
                  >
                    Remove
                  </button>
                </div>
              </div>
            {/if}
          </div>
          <input
            bind:this={logoInput}
            on:change={onLogoChange}
            type="file"
            accept="image/*"
            class="sr-only"
          />
        </fieldset>

      </div>

      <aside
        class="grid content-start justify-items-center px-6 py-10 text-center min-[760px]:px-12 min-[900px]:py-10 min-[1120px]:px-[60px]"
        aria-live="polite"
        aria-busy={busy}
      >
        <h2 class="mb-12 justify-self-start text-[15px] font-[900] uppercase tracking-[0.28em] min-[900px]:mb-24">
          Preview
        </h2>
        {#if qrState.kind === 'ready'}
          <img
            src={qrState.pngDataUrl}
            alt="Generated QR code"
            class="checkerboard aspect-square w-[min(100%,300px)] p-4"
          />
          <div class="mt-6 grid w-[min(100%,300px)] grid-cols-1 border border-text min-[520px]:grid-cols-2 min-[900px]:grid-cols-1 min-[1180px]:grid-cols-2">
            <a
              class="focus-ring inline-flex min-h-14 cursor-pointer items-center justify-center bg-accent px-4 text-[16px] font-[900] text-panel no-underline transition-colors duration-[180ms] hover:bg-accent-strong"
              href={qrState.pngDataUrl}
              download={buildFilename(qrState.text, 'png')}
            >
              Download PNG
            </a>
            <a
              class="focus-ring inline-flex min-h-14 cursor-pointer items-center justify-center border-t border-text bg-panel px-4 text-[16px] font-[900] text-text no-underline transition-colors duration-[180ms] hover:bg-text hover:text-panel min-[520px]:border-l min-[520px]:border-t-0 min-[900px]:border-l-0 min-[900px]:border-t min-[1180px]:border-l min-[1180px]:border-t-0"
              href={buildSvgDataUrl(qrState.svgMarkup)}
              download={buildFilename(qrState.text, 'svg')}
            >
              Download SVG
            </a>
          </div>
          <p class="mt-6 text-[15px] font-bold uppercase tracking-[0.22em] text-subtle">512 × 512 px</p>
          {#if busy}
            <p class="m-0 mt-3 text-sm font-semibold text-muted [overflow-wrap:anywhere]">Updating preview...</p>
          {/if}
        {:else if qrState.kind === 'generating'}
          <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-accent bg-accent-soft p-5 font-bold text-accent">
            Generating
          </div>
          <p class="m-0 mt-6 text-sm font-semibold text-muted [overflow-wrap:anywhere]">Encoding {qrState.text.length} characters</p>
        {:else if qrState.kind === 'failed'}
          <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-error bg-error-soft p-5 font-bold text-error">
            Failed
          </div>
          <p class="m-0 mt-6 text-sm font-semibold text-error [overflow-wrap:anywhere]">{qrState.message}</p>
        {:else}
          <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-dashed border-border-strong bg-panel-soft p-5 font-bold text-muted">
            Enter content
          </div>
          <p class="m-0 mt-6 text-sm font-semibold text-muted [overflow-wrap:anywhere]">Downloads unlock after a valid QR code is generated.</p>
        {/if}
        {#if qrState.kind !== 'ready'}
          <div class="mt-6 grid w-[min(100%,300px)] grid-cols-1 border border-text min-[520px]:grid-cols-2 min-[900px]:grid-cols-1 min-[1180px]:grid-cols-2">
            <button
              type="button"
              disabled
              title="Enter content to generate a QR code"
              class="inline-flex min-h-14 cursor-not-allowed items-center justify-center bg-panel-soft px-4 text-[16px] font-[900] text-subtle"
            >
              Download PNG
            </button>
            <button
              type="button"
              disabled
              title="Enter content to generate a QR code"
              class="inline-flex min-h-14 cursor-not-allowed items-center justify-center border-t border-text bg-panel px-4 text-[16px] font-[900] text-subtle min-[520px]:border-l min-[520px]:border-t-0 min-[900px]:border-l-0 min-[900px]:border-t min-[1180px]:border-l min-[1180px]:border-t-0"
            >
              Download SVG
            </button>
          </div>
        {/if}
      </aside>
    </div>
    {/if}

    <footer class="flex items-center justify-between border-t border-text px-6 py-5 text-[15px] font-bold uppercase tracking-[0.18em] text-subtle min-[760px]:px-12 min-[1120px]:px-[90px]">
      <span>&copy; 2026</span>
      <a
        href="https://github.com/zulfaza/qr-gen"
        target="_blank"
        rel="noreferrer"
        class="text-accent no-underline hover:text-accent-strong"
      >
        GitHub ↗
      </a>
    </footer>
  </section>
</main>
