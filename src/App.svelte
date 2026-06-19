<script lang="ts">
  import { toDataURL, toString } from 'qrcode'
  import type { QRCodeToDataURLOptions, QRCodeToStringOptions } from 'qrcode'

  type DownloadFormat = 'png' | 'svg'

  type QrState =
    | { kind: 'idle' }
    | { kind: 'generating'; text: string }
    | { kind: 'ready'; text: string; pngDataUrl: string; svgMarkup: string }
    | { kind: 'failed'; message: string }

  const pngOptions: QRCodeToDataURLOptions = {
    type: 'image/png',
    errorCorrectionLevel: 'M',
    margin: 2,
    width: 960,
    color: {
      dark: '#111827ff',
      light: '#ffffffff',
    },
  }

  const svgOptions: QRCodeToStringOptions = {
    type: 'svg',
    errorCorrectionLevel: 'M',
    margin: 2,
    width: 512,
    color: {
      dark: '#111827ff',
      light: '#ffffffff',
    },
  }

  let input = 'https://svelte.dev'
  let qrState: QrState = { kind: 'idle' }
  let requestId = 0

  $: void renderQr(input)

  async function renderQr(rawText: string): Promise<void> {
    const text = rawText.trim()
    const currentRequestId = requestId + 1
    requestId = currentRequestId

    if (text.length === 0) {
      qrState = { kind: 'idle' }
      return
    }

    qrState = { kind: 'generating', text }

    try {
      const [pngDataUrl, svgMarkup] = await Promise.all([
        toDataURL(text, pngOptions),
        toString(text, svgOptions),
      ])

      if (currentRequestId !== requestId) {
        return
      }

      qrState = { kind: 'ready', text, pngDataUrl, svgMarkup }
    } catch (error: unknown) {
      if (currentRequestId !== requestId) {
        return
      }

      qrState = {
        kind: 'failed',
        message: error instanceof Error ? error.message : 'QR generation failed.',
      }
    }
  }

  function downloadPng(): void {
    if (qrState.kind !== 'ready') {
      return
    }

    downloadUrl(qrState.pngDataUrl, buildFilename(qrState.text, 'png'))
  }

  function downloadSvg(): void {
    if (qrState.kind !== 'ready') {
      return
    }

    const blob = new Blob([qrState.svgMarkup], { type: 'image/svg+xml;charset=utf-8' })
    const url = URL.createObjectURL(blob)
    downloadUrl(url, buildFilename(qrState.text, 'svg'))
    URL.revokeObjectURL(url)
  }

  function downloadUrl(url: string, filename: string): void {
    const link = document.createElement('a')
    link.href = url
    link.download = filename
    link.rel = 'noopener'
    link.click()
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
</script>

<main class="shell">
  <section class="workspace" aria-labelledby="page-title">
    <div class="intro">
      <p class="eyebrow">QR Generator</p>
      <h1 id="page-title">String to QR</h1>
      <p class="summary">Generate a QR code from any text, URL, or identifier.</p>
    </div>

    <div class="generator">
      <label for="qr-input">Content</label>
      <textarea
        id="qr-input"
        bind:value={input}
        rows="7"
        spellcheck="false"
        placeholder="Paste text or URL"
      ></textarea>

      <div class="actions">
        <button type="button" onclick={downloadPng} disabled={qrState.kind !== 'ready'}>
          Download PNG
        </button>
        <button type="button" class="secondary" onclick={downloadSvg} disabled={qrState.kind !== 'ready'}>
          Download SVG
        </button>
      </div>
    </div>

    <aside class="preview" aria-live="polite">
      {#if qrState.kind === 'ready'}
        <img src={qrState.pngDataUrl} alt="Generated QR code" />
        <p>{qrState.text.length} characters encoded</p>
      {:else if qrState.kind === 'generating'}
        <div class="placeholder">Generating</div>
        <p>{qrState.text.length} characters</p>
      {:else if qrState.kind === 'failed'}
        <div class="placeholder error">Failed</div>
        <p>{qrState.message}</p>
      {:else}
        <div class="placeholder">Enter content</div>
        <p>Downloads unlock after generation.</p>
      {/if}
    </aside>
  </section>
</main>
