<script lang="ts">
  import { onDestroy, onMount } from 'svelte'
  import { BrowserQRCodeReader } from '@zxing/browser'

  type ScanMode = 'camera' | 'upload'
  type ScanState =
    | { kind: 'idle' }
    | { kind: 'scanning' }
    | { kind: 'success'; text: string }
    | { kind: 'error'; message: string }
  type NoticeState =
    | { kind: 'hidden' }
    | { kind: 'visible'; message: string }

  export let onEditQr: (text: string) => void = () => {}

  let mode: ScanMode = 'upload'
  let scanState: ScanState = { kind: 'idle' }
  let videoElement: HTMLVideoElement | undefined
  let fileInput: HTMLInputElement | undefined
  let uploadPreview = ''
  let cameraControls: { stop: () => void } | null = null
  let codeReader: BrowserQRCodeReader | null = null
  let cameraSession = 0
  let copied = false
  let copyTimer: ReturnType<typeof setTimeout> | undefined
  let notice: NoticeState = { kind: 'hidden' }
  let noticeTimer: ReturnType<typeof setTimeout> | undefined

  onMount(() => {
    window.addEventListener('paste', onPaste)
    return () => window.removeEventListener('paste', onPaste)
  })

  onDestroy(() => {
    stopCamera()
    revokeUploadPreview()
    if (copyTimer) {
      clearTimeout(copyTimer)
    }
    if (noticeTimer) {
      clearTimeout(noticeTimer)
    }
  })

  function revokeUploadPreview(): void {
    if (uploadPreview.startsWith('blob:')) {
      URL.revokeObjectURL(uploadPreview)
    }
    uploadPreview = ''
  }

  function stopVideoTracks(video: HTMLVideoElement | undefined): void {
    if (!video) {
      return
    }

    const stream = video.srcObject
    if (stream instanceof MediaStream) {
      for (const track of stream.getTracks()) {
        track.stop()
      }
    }

    video.srcObject = null
    video.pause()
  }

  function stopCamera(): void {
    cameraSession += 1
    cameraControls?.stop()
    cameraControls = null
    stopVideoTracks(videoElement)
  }

  async function startCamera(): Promise<void> {
    const video = videoElement
    if (!video || mode !== 'camera') {
      return
    }

    stopCamera()
    const session = cameraSession
    scanState = { kind: 'scanning' }

    try {
      codeReader ??= new BrowserQRCodeReader()
      const controls = await codeReader.decodeFromVideoDevice(undefined, video, (result) => {
        if (result && session === cameraSession && mode === 'camera') {
          scanState = { kind: 'success', text: result.getText() }
        }
      })

      if (session !== cameraSession || mode !== 'camera') {
        controls.stop()
        stopVideoTracks(video)
        return
      }

      cameraControls = controls
    } catch (error: unknown) {
      if (session !== cameraSession) {
        return
      }

      scanState = {
        kind: 'error',
        message: error instanceof Error ? error.message : 'Could not access the camera.',
      }
    }
  }

  async function switchMode(nextMode: ScanMode): Promise<void> {
    if (mode === nextMode) {
      return
    }

    copied = false

    if (nextMode === 'upload') {
      stopCamera()
      mode = nextMode
      scanState = { kind: 'idle' }
      return
    }

    stopCamera()
    mode = nextMode
    revokeUploadPreview()
    if (fileInput) {
      fileInput.value = ''
    }
    scanState = { kind: 'idle' }
    await startCamera()
  }

  function initVideo(node: HTMLVideoElement): { destroy: () => void } {
    videoElement = node
    if (mode === 'camera') {
      void startCamera()
    }
    return {
      destroy() {
        stopCamera()
        videoElement = undefined
      },
    }
  }

  function openFilePicker(): void {
    fileInput?.click()
  }

  async function onFileChange(event: Event): Promise<void> {
    const file = getImageFile(event.target)
    if (!file) {
      return
    }

    await decodeImageFile(file)
  }

  async function decodeImageFile(file: File): Promise<void> {
    copied = false
    hideNotice()
    revokeUploadPreview()
    if (fileInput) {
      fileInput.value = ''
    }
    uploadPreview = URL.createObjectURL(file)
    scanState = { kind: 'scanning' }

    try {
      codeReader ??= new BrowserQRCodeReader()
      const result = await codeReader.decodeFromImageUrl(uploadPreview)
      scanState = { kind: 'success', text: result.getText() }
    } catch {
      const message = 'No QR code found in this image.'
      scanState = { kind: 'error', message }
      showNotice(message)
    }
  }

  function onPaste(event: ClipboardEvent): void {
    if (mode !== 'upload') {
      return
    }

    const file = getClipboardImageFile(event.clipboardData)
    if (!file) {
      event.preventDefault()
      showNotice('Paste an image file to scan a QR code.')
      return
    }

    event.preventDefault()
    void decodeImageFile(file)
  }

  function getImageFile(target: EventTarget | null): File | undefined {
    if (!(target instanceof HTMLInputElement)) {
      return undefined
    }

    return target.files?.[0]
  }

  function getClipboardImageFile(clipboardData: DataTransfer | null): File | undefined {
    if (!clipboardData) {
      return undefined
    }

    for (const item of clipboardData.items) {
      if (item.kind !== 'file' || !item.type.startsWith('image/')) {
        continue
      }

      const file = item.getAsFile()
      if (file) {
        return file
      }
    }

    for (const file of clipboardData.files) {
      if (file.type.startsWith('image/')) {
        return file
      }
    }

    return undefined
  }

  function showNotice(message: string): void {
    if (noticeTimer) {
      clearTimeout(noticeTimer)
    }

    notice = { kind: 'visible', message }
    noticeTimer = setTimeout(hideNotice, 3500)
  }

  function hideNotice(): void {
    if (noticeTimer) {
      clearTimeout(noticeTimer)
      noticeTimer = undefined
    }

    notice = { kind: 'hidden' }
  }

  function clearUpload(): void {
    copied = false
    hideNotice()
    revokeUploadPreview()
    if (fileInput) {
      fileInput.value = ''
    }
    scanState = { kind: 'idle' }
  }

  async function copyResult(text: string): Promise<void> {
    try {
      await navigator.clipboard.writeText(text)
      copied = true
      if (copyTimer) {
        clearTimeout(copyTimer)
      }
      copyTimer = setTimeout(() => {
        copied = false
      }, 2000)
    } catch {
      copied = false
    }
  }

  function scanAgain(): void {
    copied = false
    hideNotice()
    if (mode === 'camera') {
      scanState = { kind: 'scanning' }
      void startCamera()
      return
    }

    clearUpload()
  }
</script>

<div class="grid flex-1 min-[900px]:grid-cols-[minmax(0,1fr)_39%]">
  {#if notice.kind === 'visible'}
    <div
      class="fixed right-4 top-4 z-50 flex max-w-[calc(100vw-2rem)] items-start gap-4 border border-error bg-error-soft p-4 text-left shadow-[6px_6px_0_var(--color-text)] min-[560px]:max-w-[360px]"
      role="alert"
    >
      <p class="m-0 flex-1 text-sm font-bold leading-[1.45] text-error [overflow-wrap:anywhere]">
        {notice.message}
      </p>
      <button
        type="button"
        class="focus-ring cursor-pointer border border-error bg-panel px-2 py-1 text-xs font-[900] uppercase tracking-[0.12em] text-error hover:bg-error hover:text-panel"
        on:click={hideNotice}
      >
        Close
      </button>
    </div>
  {/if}

  <div class="border-b border-text px-6 py-10 min-[760px]:px-12 min-[900px]:border-b-0 min-[900px]:border-r min-[1120px]:px-[90px] min-[1120px]:py-10">
    <div class="flex flex-wrap gap-2" role="tablist" aria-label="Scan method">
      <button
        type="button"
        role="tab"
        aria-selected={mode === 'camera'}
        class="focus-ring cursor-pointer border px-5 py-3 text-[14px] font-[900] uppercase tracking-[0.14em] transition-colors duration-[180ms] {mode === 'camera'
          ? 'border-text bg-accent text-panel'
          : 'border-border-strong bg-panel text-muted hover:border-text hover:text-text'}"
        on:click={() => switchMode('camera')}
      >
        Camera
      </button>
      <button
        type="button"
        role="tab"
        aria-selected={mode === 'upload'}
        class="focus-ring cursor-pointer border px-5 py-3 text-[14px] font-[900] uppercase tracking-[0.14em] transition-colors duration-[180ms] {mode === 'upload'
          ? 'border-text bg-accent text-panel'
          : 'border-border-strong bg-panel text-muted hover:border-text hover:text-text'}"
        on:click={() => switchMode('upload')}
      >
        Upload image
      </button>
    </div>

    {#if mode === 'camera'}
      <div class="mt-8" role="tabpanel">
        <p class="text-[17px] font-semibold text-subtle">
          Point your camera at a QR code. The result appears as soon as one is detected.
        </p>
        <div class="mt-6 overflow-hidden border border-text bg-text">
          <video
            use:initVideo
            class="aspect-square w-full max-w-[520px] object-cover"
            muted
            playsinline
            aria-label="Camera preview for QR scanning"
          ></video>
        </div>
      </div>
    {:else}
      <div class="mt-8" role="tabpanel">
        <p class="text-[17px] font-semibold text-subtle">
          Upload or paste a photo or screenshot that contains a QR code.
        </p>
        <p class="mt-4 text-[17px] font-semibold text-subtle">
          Supported formats: JPG, PNG, WebP, GIF, and BMP.
        </p>
        <div class="mt-6 flex flex-wrap items-start gap-6">
          {#if uploadPreview}
            <img
              src={uploadPreview}
              alt=""
              class="aspect-square w-[min(100%,220px)] border border-text object-cover"
            />
            <div class="flex flex-col gap-3">
              <button
                type="button"
                class="focus-ring cursor-pointer border border-text bg-panel px-4 py-2 text-sm font-[900] uppercase tracking-[0.12em] text-text hover:bg-accent hover:text-panel"
                on:click={openFilePicker}
              >
                Replace
              </button>
              <button
                type="button"
                class="focus-ring cursor-pointer border border-border-strong bg-panel px-4 py-2 text-sm font-[900] uppercase tracking-[0.12em] text-muted hover:border-text hover:text-text"
                on:click={clearUpload}
              >
                Remove
              </button>
            </div>
          {:else}
            <button
              type="button"
              class="focus-ring grid aspect-square w-[min(100%,220px)] cursor-pointer place-items-center border border-dashed border-border-strong bg-panel-soft p-6 text-center hover:border-accent"
              on:click={openFilePicker}
            >
              <span class="text-[38px] font-light leading-none text-subtle">+</span>
              <span class="mt-3 text-[15px] font-[900] uppercase tracking-[0.16em] text-muted">Choose image</span>
            </button>
          {/if}
        </div>
        <input
          bind:this={fileInput}
          on:change={onFileChange}
          type="file"
          accept="image/jpeg,image/png,image/webp,image/gif,image/bmp,image/*"
          class="sr-only"
        />
      </div>
    {/if}
  </div>

  <aside
    class="grid content-start justify-items-center px-6 py-10 text-center min-[760px]:px-12 min-[900px]:py-10 min-[1120px]:px-[60px]"
    aria-live="polite"
    aria-busy={scanState.kind === 'scanning'}
  >
    <h2 class="mb-12 justify-self-start text-[15px] font-[900] uppercase tracking-[0.28em] min-[900px]:mb-24">
      Result
    </h2>

    {#if scanState.kind === 'success'}
      {@const decodedText = scanState.text}
      <div class="w-full max-w-[420px] border border-text bg-panel-soft p-6 text-left">
        <p class="m-0 text-[15px] font-[900] uppercase tracking-[0.22em] text-subtle">Decoded content</p>
        <p class="m-0 mt-4 break-all text-[20px] font-semibold leading-[1.4] text-text [overflow-wrap:anywhere]">
          {decodedText}
        </p>
      </div>
      <div class="mt-6 grid w-[min(100%,420px)] grid-cols-1 border border-text min-[520px]:grid-cols-2 min-[900px]:grid-cols-1 min-[1180px]:grid-cols-2">
        <button
          type="button"
          class="focus-ring inline-flex min-h-14 cursor-pointer items-center justify-center bg-accent px-4 text-[16px] font-[900] text-panel transition-colors duration-[180ms] hover:bg-accent-strong"
          on:click={() => copyResult(decodedText)}
        >
          {copied ? 'Copied' : 'Copy text'}
        </button>
        <button
          type="button"
          class="focus-ring inline-flex min-h-14 cursor-pointer items-center justify-center border-t border-text bg-panel px-4 text-[16px] font-[900] text-text transition-colors duration-[180ms] hover:bg-text hover:text-panel min-[520px]:border-l min-[520px]:border-t-0 min-[900px]:border-l-0 min-[900px]:border-t min-[1180px]:border-l min-[1180px]:border-t-0"
          on:click={() => onEditQr(decodedText)}
        >
          Edit QR
        </button>
      </div>
      {#if decodedText.startsWith('http://') || decodedText.startsWith('https://')}
        <a
          href={decodedText}
          target="_blank"
          rel="noreferrer"
          class="focus-ring mt-6 text-[15px] font-bold uppercase tracking-[0.18em] text-accent no-underline hover:text-accent-strong"
        >
          Open link ↗
        </a>
      {/if}
    {:else if scanState.kind === 'scanning'}
      <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-accent bg-accent-soft p-5 font-bold text-accent">
        Scanning
      </div>
      <p class="m-0 mt-6 text-sm font-semibold text-muted [overflow-wrap:anywhere]">
        {mode === 'camera' ? 'Looking for a QR code in the camera feed...' : 'Reading the uploaded image...'}
      </p>
    {:else if scanState.kind === 'error'}
      <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-error bg-error-soft p-5 font-bold text-error">
        Failed
      </div>
      <p class="m-0 mt-6 text-sm font-semibold text-error [overflow-wrap:anywhere]">{scanState.message}</p>
      <button
        type="button"
        class="focus-ring mt-6 cursor-pointer border border-text bg-panel px-4 py-2 text-sm font-[900] uppercase tracking-[0.12em] text-text hover:bg-accent hover:text-panel"
        on:click={scanAgain}
      >
        Try again
      </button>
    {:else}
      <div class="grid aspect-square w-[min(100%,300px)] place-items-center border border-dashed border-border-strong bg-panel-soft p-5 font-bold text-muted">
        Waiting
      </div>
      <p class="m-0 mt-6 text-sm font-semibold text-muted [overflow-wrap:anywhere]">
        {mode === 'camera' ? 'Start the camera and aim it at a QR code.' : 'Upload an image to decode its QR code.'}
      </p>
    {/if}
  </aside>
</div>
