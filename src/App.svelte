<script lang="ts">
  import { toDataURL, toString } from 'qrcode'
  import type { QRCodeToDataURLOptions, QRCodeToStringOptions } from 'qrcode'
  import ColorPicker from './ColorPicker.svelte'

  type DownloadFormat = 'png' | 'svg'

  type QrState =
    | { kind: 'idle' }
    | { kind: 'generating'; text: string }
    | { kind: 'ready'; text: string; pngDataUrl: string; svgMarkup: string }
    | { kind: 'failed'; message: string }

  let input = 'hello there'
  let darkColor = '#111827ff'
  let lightColor = '#ffffffff'
  let qrState: QrState = { kind: 'idle' }
  let busy = false
  let requestId = 0

  $: void renderQr(input, darkColor, lightColor)
  $: encodedLength = getEncodedLength(qrState, input)

  function buildPngOptions(dark: string, light: string): QRCodeToDataURLOptions {
    return {
      type: 'image/png',
      errorCorrectionLevel: 'M',
      margin: 2,
      width: 960,
      color: { dark, light },
    }
  }

  function buildSvgOptions(dark: string, light: string): QRCodeToStringOptions {
    return {
      type: 'svg',
      errorCorrectionLevel: 'M',
      margin: 2,
      width: 512,
      color: { dark, light },
    }
  }

  async function renderQr(rawText: string, dark: string, light: string): Promise<void> {
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
      const [pngDataUrl, svgMarkup] = await Promise.all([
        toDataURL(text, buildPngOptions(dark, light)),
        toString(text, buildSvgOptions(dark, light)),
      ])

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
</script>

<main class="shell">
  <section class="workspace" aria-labelledby="page-title">
    <header class="intro">
      <p class="utility-label">Personal QR utility</p>
      <h1 id="page-title">QR generator</h1>
      <p class="summary">Paste text or a URL. Preview updates automatically; PNG and SVG unlock when ready.</p>
    </header>

    <div class="generator">
      <div class="field-header">
        <label for="qr-input">Content</label>
        <span>{encodedLength} characters</span>
      </div>
      <textarea
        id="qr-input"
        bind:value={input}
        rows="7"
        spellcheck="false"
        placeholder="Paste text or URL"
        aria-describedby="qr-input-hint"
      ></textarea>
      <p id="qr-input-hint" class="field-hint">Leading and trailing whitespace is trimmed before encoding.</p>

      <fieldset class="colors">
        <legend>Colors</legend>
        <div class="color-row">
          <div class="color-field">
            <span>QR color</span>
            <ColorPicker bind:value={darkColor} label="QR color" />
          </div>
          <div class="color-field">
            <span>Background</span>
            <ColorPicker bind:value={lightColor} label="Background color" />
          </div>
        </div>
        <p class="field-hint">Lower a color's opacity for a transparent background (PNG only).</p>
      </fieldset>

      <div class="actions">
        {#if qrState.kind === 'ready'}
          <a class="download-button" href={qrState.pngDataUrl} download={buildFilename(qrState.text, 'png')}>
            Download PNG
          </a>
          <a
            class="download-button secondary"
            href={buildSvgDataUrl(qrState.svgMarkup)}
            download={buildFilename(qrState.text, 'svg')}
          >
            Download SVG
          </a>
        {:else}
          <button type="button" disabled title="Enter content to generate a QR code">Download PNG</button>
          <button type="button" class="secondary" disabled title="Enter content to generate a QR code">
            Download SVG
          </button>
        {/if}
      </div>
    </div>

    <aside class="preview" aria-live="polite" aria-busy={busy}>
      {#if qrState.kind === 'ready'}
        <img src={qrState.pngDataUrl} alt="Generated QR code" />
        {#if busy}
          <p class="status">Updating preview…</p>
        {:else}
          <p class="status ready">Ready to download · {qrState.text.length} characters encoded</p>
        {/if}
      {:else if qrState.kind === 'generating'}
        <div class="placeholder generating">Generating</div>
        <p class="status">Encoding {qrState.text.length} characters</p>
      {:else if qrState.kind === 'failed'}
        <div class="placeholder error">Failed</div>
        <p class="status error">{qrState.message}</p>
      {:else}
        <div class="placeholder">Enter content</div>
        <p class="status">Downloads unlock after a valid QR code is generated.</p>
      {/if}
    </aside>
  </section>
</main>
