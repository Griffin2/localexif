<script>
  import TrustBar from "$lib/TrustBar.svelte";
  import ToolHeader from "$lib/ToolHeader.svelte";
  import ToolFooter from "$lib/ToolFooter.svelte";
  import ExifReader from "exifreader";

  let files = $state([]);
  let processing = $state(false);
  let dragOver = $state(false);
  let expandedId = $state(null);

  function handleDrop(e) {
    e.preventDefault();
    dragOver = false;
    const dropped = Array.from(e.dataTransfer.files).filter((f) =>
      f.type.startsWith("image/"),
    );
    if (dropped.length) addFiles(dropped);
  }

  function handleSelect(e) {
    const selected = Array.from(e.target.files);
    if (selected.length) addFiles(selected);
    e.target.value = "";
  }

  async function addFiles(newFiles) {
    for (const file of newFiles) {
      const entry = {
        id: crypto.randomUUID(),
        name: file.name,
        type: file.type,
        originalSize: file.size,
        cleanedSize: null,
        cleanedUrl: null,
        status: "pending",
        metadata: null,
        metadataCount: 0,
        hasGps: false,
      };
      files = [...files, entry];
      await processFile(file, entry.id);
    }
  }

  function parseMetadata(arrayBuffer) {
    try {
      const tags = ExifReader.load(arrayBuffer, { expanded: true });
      const entries = [];
      let hasGps = false;

      const PRIORITY_TAGS = [
        "GPSLatitude",
        "GPSLongitude",
        "GPSAltitude",
        "DateTimeOriginal",
        "CreateDate",
        "ModifyDate",
        "Make",
        "Model",
        "LensModel",
        "Software",
        "Artist",
        "Copyright",
        "ImageWidth",
        "ImageHeight",
        "ExposureTime",
        "FNumber",
        "ISO",
        "FocalLength",
      ];

      const seen = new Set();

      function addTag(category, name, tag) {
        if (!tag || seen.has(name)) return;
        seen.add(name);
        const val = tag.description || tag.value;
        if (val === undefined || val === null || val === "") return;
        entries.push({ category, name, value: String(val) });
      }

      // GPS
      if (tags.gps) {
        hasGps = true;
        if (tags.gps.Latitude !== undefined) {
          entries.push({
            category: "GPS",
            name: "Latitude",
            value: String(tags.gps.Latitude),
          });
          seen.add("GPSLatitude");
        }
        if (tags.gps.Longitude !== undefined) {
          entries.push({
            category: "GPS",
            name: "Longitude",
            value: String(tags.gps.Longitude),
          });
          seen.add("GPSLongitude");
        }
        if (tags.gps.Altitude !== undefined) {
          entries.push({
            category: "GPS",
            name: "Altitude",
            value: `${tags.gps.Altitude}m`,
          });
          seen.add("GPSAltitude");
        }
      }

      // EXIF
      if (tags.exif) {
        for (const [key, tag] of Object.entries(tags.exif)) {
          addTag("EXIF", key, tag);
        }
      }

      // File / IFD0
      if (tags.file) {
        for (const [key, tag] of Object.entries(tags.file)) {
          addTag("File", key, tag);
        }
      }

      // IPTC
      if (tags.iptc) {
        for (const [key, tag] of Object.entries(tags.iptc)) {
          addTag("IPTC", key, tag);
        }
      }

      // XMP
      if (tags.xmp) {
        for (const [key, tag] of Object.entries(tags.xmp)) {
          addTag("XMP", key, tag);
        }
      }

      // Sort: priority tags first
      const prioritySet = new Set(PRIORITY_TAGS);
      entries.sort((a, b) => {
        const aPri = prioritySet.has(a.name) ? 0 : 1;
        const bPri = prioritySet.has(b.name) ? 0 : 1;
        if (aPri !== bPri) return aPri - bPri;
        // GPS category first
        if (a.category === "GPS" && b.category !== "GPS") return -1;
        if (b.category === "GPS" && a.category !== "GPS") return 1;
        return 0;
      });

      return { entries, hasGps };
    } catch {
      return { entries: [], hasGps: false };
    }
  }

  function stripViaCanvas(file) {
    return new Promise((resolve, reject) => {
      const url = URL.createObjectURL(file);
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement("canvas");
        canvas.width = img.naturalWidth;
        canvas.height = img.naturalHeight;
        const ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0);
        URL.revokeObjectURL(url);

        const outputType =
          file.type === "image/png" ? "image/png" : "image/jpeg";
        const quality = file.type === "image/png" ? undefined : 0.95;

        canvas.toBlob(
          (blob) => {
            if (blob) resolve(blob);
            else reject(new Error("Canvas export failed"));
          },
          outputType,
          quality,
        );
      };
      img.onerror = () => {
        URL.revokeObjectURL(url);
        reject(new Error("Image load failed"));
      };
      img.src = url;
    });
  }

  async function processFile(file, id) {
    processing = true;
    try {
      // Read metadata first
      const arrayBuffer = await file.arrayBuffer();
      const { entries, hasGps } = parseMetadata(arrayBuffer);

      // Then strip via canvas
      const blob = await stripViaCanvas(file);
      const url = URL.createObjectURL(blob);

      files = files.map((f) =>
        f.id === id
          ? {
              ...f,
              cleanedSize: blob.size,
              cleanedUrl: url,
              status: "done",
              metadata: entries,
              metadataCount: entries.length,
              hasGps: hasGps,
            }
          : f,
      );
    } catch (e) {
      files = files.map((f) => (f.id === id ? { ...f, status: "error" } : f));
    }
    processing = false;
  }

  function toggleExpand(id) {
    expandedId = expandedId === id ? null : id;
  }

  function downloadFile(entry) {
    if (!entry.cleanedUrl) return;
    const a = document.createElement("a");
    a.href = entry.cleanedUrl;
    a.download = `clean-${entry.name}`;
    a.click();
  }

  function downloadAll() {
    files.filter((f) => f.status === "done").forEach(downloadFile);
  }

  function clear() {
    files.forEach((f) => {
      if (f.cleanedUrl) URL.revokeObjectURL(f.cleanedUrl);
    });
    files = [];
    expandedId = null;
  }

  function formatSize(bytes) {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1048576) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / 1048576).toFixed(1)} MB`;
  }

  function getCategoryColor(cat) {
    switch (cat) {
      case "GPS":
        return "#dc2626";
      case "EXIF":
        return "#e8a830";
      case "IPTC":
        return "#9b8fde";
      case "XMP":
        return "#6baf8d";
      default:
        return "#999";
    }
  }

  let doneCount = $derived(files.filter((f) => f.status === "done").length);
  let totalSaved = $derived(
    files
      .filter((f) => f.status === "done")
      .reduce((sum, f) => sum + (f.originalSize - f.cleanedSize), 0),
  );
  let totalGps = $derived(files.filter((f) => f.hasGps).length);
</script>

<TrustBar sourceUrl="https://github.com/user/localexif" />

<div class="page-shell">
  <ToolHeader
    title="EXIF Stripper"
    description="Remove GPS, camera data & metadata from photos — nothing leaves your browser."
  />

  <div
    class="drop-zone"
    class:drag-over={dragOver}
    ondragover={(e) => {
      e.preventDefault();
      dragOver = true;
    }}
    ondragleave={() => (dragOver = false)}
    ondrop={handleDrop}
    role="button"
    tabindex="0"
  >
    <div class="drop-content">
      <span class="drop-icon">📷</span>
      <p class="drop-text">Drop images here or click to select</p>
      <p class="drop-hint">
        JPEG, PNG, WebP — processed entirely in your browser
      </p>
      <label class="btn btn-primary drop-btn">
        Select Images
        <input
          type="file"
          accept="image/jpeg,image/png,image/webp"
          multiple
          onchange={handleSelect}
          hidden
        />
      </label>
    </div>
  </div>

  {#if files.length > 0}
    <div class="controls">
      <button
        class="btn btn-primary"
        onclick={downloadAll}
        disabled={doneCount === 0}
      >
        Download All ({doneCount})
      </button>
      <button class="btn" onclick={clear}>Clear</button>
      <div class="spacer"></div>
      {#if totalGps > 0}
        <span class="pill pill-gps">
          <span class="pill-dot-gps"></span>
          {totalGps} with GPS
        </span>
      {/if}
      {#if doneCount > 0}
        <span class="pill pill-ok">
          <span class="pill-dot"></span>
          {totalSaved > 0
            ? formatSize(totalSaved) + " stripped"
            : "metadata cleared"}
        </span>
      {/if}
    </div>

    <div class="file-list">
      {#each files as entry}
        <div class="file-entry">
          <div
            class="file-row"
            class:file-done={entry.status === "done"}
            class:file-error={entry.status === "error"}
            class:file-has-gps={entry.hasGps}
          >
            <span class="file-name">
              {entry.name}
              {#if entry.hasGps}
                <span class="gps-badge" title="Contains GPS coordinates"
                  >📍</span
                >
              {/if}
            </span>
            <span class="file-meta-count">
              {#if entry.metadataCount > 0}
                <button
                  class="meta-toggle"
                  onclick={() => toggleExpand(entry.id)}
                  title="View found metadata"
                >
                  {entry.metadataCount} tag{entry.metadataCount !== 1
                    ? "s"
                    : ""}
                  {expandedId === entry.id ? "▴" : "▾"}
                </button>
              {/if}
            </span>
            <span class="file-sizes">
              {formatSize(entry.originalSize)}
              {#if entry.cleanedSize !== null}
                <span class="file-arrow">→</span>
                {formatSize(entry.cleanedSize)}
              {/if}
            </span>
            <span class="file-status">
              {#if entry.status === "pending"}
                <span class="spinner"></span>
              {:else if entry.status === "done"}
                <button class="btn btn-sm" onclick={() => downloadFile(entry)}>
                  Download
                </button>
              {:else}
                <span class="file-err-text">unsupported</span>
              {/if}
            </span>
          </div>

          {#if expandedId === entry.id && entry.metadata && entry.metadata.length > 0}
            <div class="meta-panel">
              {#each entry.metadata as tag}
                <div
                  class="meta-row"
                  class:meta-row-gps={tag.category === "GPS"}
                >
                  <span
                    class="meta-cat"
                    style="color: {getCategoryColor(tag.category)};"
                    >{tag.category}</span
                  >
                  <span class="meta-name">{tag.name}</span>
                  <span
                    class="meta-value"
                    class:meta-value-gps={tag.category === "GPS"}
                    >{tag.value}</span
                  >
                </div>
              {/each}
              <div class="meta-footer">
                All {entry.metadata.length} tags removed from cleaned file
              </div>
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}

  <ToolFooter />
</div>

<style>
  .drop-zone {
    border: 2px dashed var(--border-light);
    border-radius: var(--radius-md);
    padding: 48px 24px;
    text-align: center;
    cursor: pointer;
    transition:
      border-color 0.15s,
      background 0.15s;
    background: var(--bg-surface);
  }

  .drop-zone:hover,
  .drag-over {
    border-color: var(--accent);
    background: var(--accent-bg);
  }

  .drop-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }

  .drop-icon {
    font-size: 32px;
  }

  .drop-text {
    font-size: 14px;
    color: var(--text-primary);
    margin: 0;
  }

  .drop-hint {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-muted);
    margin: 0;
  }

  .drop-btn {
    margin-top: 8px;
    cursor: pointer;
  }

  .file-list {
    margin-top: 16px;
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    overflow: hidden;
    background: var(--bg-surface);
  }

  .file-entry {
    border-bottom: 1px solid var(--border-light);
  }

  .file-entry:last-child {
    border-bottom: none;
  }

  .file-row {
    display: flex;
    align-items: center;
    padding: 10px 14px;
    gap: 12px;
    font-family: var(--font-mono);
    font-size: 12px;
  }

  .file-has-gps {
    background: #fef2f2;
  }

  .file-name {
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    color: var(--text-primary);
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .gps-badge {
    font-size: 12px;
    flex-shrink: 0;
  }

  .file-sizes {
    color: var(--text-muted);
    white-space: nowrap;
  }

  .file-arrow {
    color: var(--accent);
    margin: 0 4px;
  }

  .file-done .file-name {
    color: var(--text-primary);
  }

  .file-error .file-name {
    color: #dc2626;
  }

  .file-err-text {
    color: #dc2626;
    font-size: 11px;
  }

  .btn-sm {
    font-size: 11px;
    padding: 4px 10px;
  }

  .meta-toggle {
    background: none;
    border: 1px solid var(--border-light);
    border-radius: var(--radius-sm);
    font-family: var(--font-mono);
    font-size: 10px;
    color: var(--text-muted);
    padding: 2px 8px;
    cursor: pointer;
    white-space: nowrap;
  }

  .meta-toggle:hover {
    border-color: var(--accent-border);
    color: var(--accent);
  }

  /* ---- metadata panel ---- */
  .meta-panel {
    background: var(--bg-input);
    border-top: 1px solid var(--border-light);
    max-height: 280px;
    overflow-y: auto;
  }

  .meta-row {
    display: flex;
    align-items: baseline;
    padding: 4px 14px;
    gap: 10px;
    font-family: var(--font-mono);
    font-size: 11px;
    border-bottom: 1px solid transparent;
  }

  .meta-row-gps {
    background: #fef2f2;
  }

  .meta-cat {
    width: 36px;
    flex-shrink: 0;
    font-size: 9px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }

  .meta-name {
    width: 140px;
    flex-shrink: 0;
    color: var(--text-secondary);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .meta-value {
    flex: 1;
    color: var(--text-muted);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .meta-value-gps {
    color: #dc2626;
    font-weight: 600;
  }

  .meta-footer {
    padding: 6px 14px;
    font-family: var(--font-mono);
    font-size: 10px;
    color: var(--text-muted);
    text-align: center;
    border-top: 1px solid var(--border-light);
    background: var(--bg-surface);
  }

  .pill-gps {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    padding: 4px 10px;
    border-radius: 999px;
    background: #fef2f2;
    color: #dc2626;
    margin-right: 6px;
  }

  .pill-dot-gps {
    display: inline-block;
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: #dc2626;
    margin-right: 4px;
  }

  .spinner {
    display: inline-block;
    width: 14px;
    height: 14px;
    border: 2px solid var(--border-light);
    border-top-color: var(--accent);
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .spacer {
    margin-left: auto;
  }

  @media (max-width: 640px) {
    .file-sizes,
    .file-meta-count {
      display: none;
    }

    .meta-name {
      width: 100px;
    }
  }
</style>
