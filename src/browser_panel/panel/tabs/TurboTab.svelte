<script>
  import { Pane, Splitpanes } from "svelte-splitpanes"
  import { slide } from "svelte/transition"
  import hljs from "highlight.js/lib/core"
  import xml from "highlight.js/lib/languages/xml"
  hljs.registerLanguage("xml", xml)

  import { getTurboFrames, getTurboStreams } from "../../State.svelte.js"
  import { inspectElement, debounce } from "../../../utils/utils.js"
  import { panelPostMessage } from "../../messaging.js"
  import { PANEL_TO_BACKEND_MESSAGES } from "../../../lib/constants.js"
  import { HOTWIRE_DEV_TOOLS_PANEL_SOURCE } from "../../ports.js"
  import { horizontalPanes } from "../../theme.svelte.js"
  import * as Icons from "../../../utils/icons.js"

  const SELECTABLE_TYPES = {
    TURBO_FRAME: "turbo-frame",
    TURBO_STREAM: "turbo-stream",
  }
  const turboStreamAnimationDuration = 300
  const ignoredAttributes = ["id", "loading", "src", "complete", "aria-busy", "busy"]

  let turboFrames = $state([])
  let turboStreams = $state([])

  let selected = $state({
    type: null,
    uuid: null,
    frame: null,
    stream: null,
  })

  const setSelectedTurboFrame = (frame) => {
    selected = {
      type: SELECTABLE_TYPES.TURBO_FRAME,
      uuid: frame.uuid,
      frame: frame,
      stream: null,
    }
  }

  const setSelectedTurboStream = (stream) => {
    selected = {
      type: SELECTABLE_TYPES.TURBO_STREAM,
      uuid: stream.uuid,
      frame: null,
      stream: stream,
    }
  }

  // Set the first Turbo Frame as selected if none is selected
  $effect(() => {
    turboFrames = getTurboFrames().sort((a, b) => a.id.localeCompare(b.id))
    turboStreams = getTurboStreams()

    if (!selected.uuid && turboFrames.length > 0) {
      selected = {
        type: SELECTABLE_TYPES.TURBO_FRAME,
        uuid: turboFrames[0].uuid,
        frame: turboFrames[0],
        stream: null,
      }
    }
  })

  const scrollIntoView = debounce((element) => {
    element.scrollIntoView({ behavior: "smooth", block: "end" })
  }, turboStreamAnimationDuration)

  const addHighlightOverlay = (selector) => {
    panelPostMessage({
      action: PANEL_TO_BACKEND_MESSAGES.HOVER_COMPONENT,
      source: HOTWIRE_DEV_TOOLS_PANEL_SOURCE,
      selector: selector,
    })
  }

  const hideHighlightOverlay = () => {
    panelPostMessage({
      action: PANEL_TO_BACKEND_MESSAGES.HIDE_HOVER,
      source: HOTWIRE_DEV_TOOLS_PANEL_SOURCE,
    })
  }

  const refreshTurboFrames = () => {
    panelPostMessage({
      action: PANEL_TO_BACKEND_MESSAGES.GET_TURBO_FRAMES,
      source: HOTWIRE_DEV_TOOLS_PANEL_SOURCE,
    })
  }

  const refreshTurboFrame = (turboFrameId) => {
    panelPostMessage({
      action: PANEL_TO_BACKEND_MESSAGES.REFRESH_TURBO_FRAME,
      source: HOTWIRE_DEV_TOOLS_PANEL_SOURCE,
      id: turboFrameId,
    })
  }

  const handleKeyDown = (event) => {
    if (!turboFrames.length) return

    const currentIndex = turboFrames.findIndex((frame) => frame.uuid === selected.uuid)
    let newIndex = currentIndex

    switch (event.key) {
      case "ArrowDown":
        event.preventDefault()
        newIndex = currentIndex < turboFrames.length - 1 ? currentIndex + 1 : 0
        break
      case "ArrowUp":
        event.preventDefault()
        newIndex = currentIndex > 0 ? currentIndex - 1 : turboFrames.length - 1
        break
      case "Home":
        event.preventDefault()
        newIndex = 0
        break
      case "End":
        event.preventDefault()
        newIndex = turboFrames.length - 1
        break
      default:
        return
    }

    selected = {
      type: SELECTABLE_TYPES.TURBO_FRAME,
      uuid: turboFrames[newIndex].uuid,
    }

    setTimeout(() => {
      const selectedRow = document.querySelector(".turbo-frames-list-panel tr.selected")
      if (selectedRow) {
        selectedRow.scrollIntoView({ behavior: "smooth", block: "center" })
      }
    }, 10)
  }
</script>

<Splitpanes class="turbo-frames-list-panel" horizontal={$horizontalPanes}>
  <Pane size={35}>
    <div class="d-flex flex-column h-100">
      <div class="d-flex justify-content-center">
        <h2>Streams</h2>
      </div>
      {#if turboStreams.length > 0}
        <div class="scrollable-list">
          {#each turboStreams as stream}
            <div
              {@attach scrollIntoView}
              class="entry-row p-1 cursor-pointer"
              transition:slide={{ duration: turboStreamAnimationDuration }}
              class:selected={selected.type === SELECTABLE_TYPES.TURBO_STREAM && selected.uuid === stream.uuid}
              role="button"
              tabindex="0"
              onclick={() => setSelectedTurboStream(stream)}
              onkeyup={() => setSelectedTurboStream(stream)}
            >
              <div class="text-align-right text-muted">
                <span class="timestamp">{stream.time}</span>
              </div>
              <div class="d-flex justify-content-between align-items-center">
                <div>{stream.action}</div>
                <div>{stream.targetSelector}</div>
              </div>
            </div>
          {/each}
        </div>
      {:else}
        <div class="no-entry-hint">
          <span>No Turbo Streams seen yet</span>
          <span>We'll keep looking</span>
        </div>
      {/if}
    </div>
  </Pane>

  <Pane size={35}>
    <div class="d-flex flex-column h-100">
      <div class="d-flex justify-content-center align-items-center position-relative">
        <h2>Frames</h2>
        <button class="btn-icon icon-dark position-absolute end-0" onclick={refreshTurboFrames} title="Refresh Turbo Frames List">
          {@html Icons.refresh}
        </button>
      </div>
      {#if turboFrames.length > 0}
        <div class="scrollable-list">
          {#each turboFrames as frame}
            {@const selector = `#${frame.id}`}
            <div
              class="p-1 d-flex justify-content-between align-items-center cursor-pointer entry-row"
              class:selected={selected.type === SELECTABLE_TYPES.TURBO_FRAME && selected.frame.id === frame.id}
              role="button"
              tabindex="0"
              onclick={() => setSelectedTurboFrame(frame)}
              onkeyup={() => setSelectedTurboFrame(frame)}
              onmouseenter={() => addHighlightOverlay(selector)}
              onmouseleave={() => hideHighlightOverlay()}
            >
              <div>{selector}</div>
              <button class="btn-icon btn-inspect" onclick={() => inspectElement(selector)}>
                {@html Icons.inspectElement}
              </button>
            </div>
          {/each}
        </div>
      {:else}
        <div class="no-entry-hint">
          <span>No Turbo Frames found on this page</span>
          <span>We'll keep looking</span>
        </div>
      {/if}
    </div>
  </Pane>

  <Pane size={30} class="turbo-frames-detail-panel flow">
    {#if selected.type === SELECTABLE_TYPES.TURBO_FRAME && selected.uuid}
      <div class="d-flex justify-content-center align-items-center position-relative">
        <h2>#{selected.frame.id}</h2>
        {#if selected.frame.src}
          <div class="position-absolute end-0">
            <button class="btn-icon icon-dark" onclick={() => refreshTurboFrame(selected.frame.id)} title="Refresh Turbo Frames List">
              {@html Icons.refresh}
            </button>
          </div>
        {/if}
      </div>

      <div class="pane-section-heading">Attributes</div>
      <table class="table table-sm w-100">
        <tbody>
          <tr>
            <td>Source</td>
            <td>{selected.frame.src || "N/A"}</td>
          </tr>
          <tr>
            <td>Loading</td>
            <td>{selected.frame.loading || "N/A"}</td>
          </tr>
          {#each Object.entries(selected.frame.attributes).filter(([key]) => !ignoredAttributes.includes(key)) as [key, value]}
            <tr>
              <td>{key}</td>
              <td>{value}</td>
            </tr>
          {/each}
        </tbody>
      </table>

      <div class="pane-section-heading">HTML</div>
      <div class="d-flex justify-content-between align-items-top gap-2">
        <div class="html-preview">
          <pre><code class="language-html">{@html hljs.highlight(selected.frame.html, { language: "html" }).value}</code></pre>
        </div>
        <button class="btn-icon btn-inspect" onclick={() => inspectElement(`#${selected.frame.id}`)}>
          {@html Icons.inspectElement}
        </button>
      </div>
    {:else if selected.type === SELECTABLE_TYPES.TURBO_STREAM && selected.uuid}
      <div class="d-flex justify-content-center align-items-center position-relative">
        <h2>{selected.stream.action} {selected.stream.targetSelector}</h2>
      </div>
      <div class="scrollable-list flow">
        <div class="pane-section-heading">Attributes</div>
        <table class="table table-sm w-100">
          <tbody>
            <tr>
              <td>Action</td>
              <td>{selected.stream.action}</td>
            </tr>
            <tr>
              <td>Target</td>
              <td>
                <div class="d-flex justify-content-between align-items-center">
                  <span>{selected.stream.targetSelector}</span>
                  <button class="btn-icon btn-inspect" onclick={() => inspectElement(`${selected.stream.targetSelector}`)}>
                    {@html Icons.inspectElement}
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>

        <div class="pane-section-heading">HTML</div>
        <div class="html-preview">
          <pre><code class="language-html">{@html hljs.highlight(selected.stream.turboStreamContent, { language: "html" }).value}</code></pre>
        </div>
      </div>
    {:else}
      <div class="no-entry-hint">
        <span>Nothing selected</span>
        <span>Select a Turbo Frame or Turbo Stream to see its details</span>
      </div>
    {/if}
  </Pane>
</Splitpanes>
