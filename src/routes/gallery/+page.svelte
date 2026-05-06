<script>
    import { onMount } from "svelte";

    // Use $state for reactive variables in Svelte runes mode
    let page = $state(1);
    let images = $state([]);
    let totalPages = $state(1); // will be calculated from the full image list
    let itemsPerPage = $state(12); // number of images per page
    let allImages = $state([]); // holds the complete list of images from the API
    // Track which image's action menu is open (null when none)
    let menuOpen = $state(null);

    async function loadPage(p) {
        try {
            // Fetch the image list for the requested page (API provides pagination info)
            const res = await fetch(`/camapi/image_gallery?page=${p}`);
            const data = await res.json();
            // Update pagination based on API response
            totalPages = data.total_pages ?? 1;
            page = data.page ?? p;
            // Build URLs for images on this page
            const files = data.image_files || [];
            images = files.map((f) => ({
                url: `/camapi/view_image/${f.filename}`,
                filename: f.filename,
                // Preserve DNG info for potential future use
                dng_file: f.dng_file,
                has_dng: f.has_dng,
            }));
            // Optionally keep a full list if needed (not required for pagination)
            allImages = [];
        } catch (e) {
            console.error("Failed to load gallery", e);
        }
    }

    function prev() {
        if (page > 1) loadPage(page - 1);
    }
    function next() {
        if (page < totalPages) loadPage(page + 1);
    }

    onMount(() => {
        loadPage(1);
    });
    // Log each image URL whenever the images array updates using $effect (runes mode)
    $effect(() => {
        // No console logging as per request
    });

    // Handle click on an image: download or delete
    // Functions for the new action menu
    function openMenu(img) {
        menuOpen = img.filename;
    }
    function closeMenu() {
        menuOpen = null;
    }
    async function downloadImage(img) {
        const a = document.createElement("a");
        a.href = img.url;
        a.download = img.filename;
        a.click();
        closeMenu();
    }
    // UI prompt state for download choice
    let downloadPrompt = $state({ open: false, img: null });

    function openDownloadPrompt(img) {
        downloadPrompt = { open: true, img };
    }

    // Decide whether to show the prompt or directly download JPG
    function handleDownloadClick(img) {
        if (img.has_dng) {
            openDownloadPrompt(img);
        } else {
            downloadImage(img);
        }
    }

    function closeDownloadPrompt() {
        downloadPrompt = { open: false, img: null };
    }

    function confirmDownloadJpg() {
        if (downloadPrompt.img) {
            downloadImage(downloadPrompt.img);
        }
        closeDownloadPrompt();
    }

    function confirmDownloadRaw() {
        if (downloadPrompt.img) {
            downloadRawImage(downloadPrompt.img);
        }
        closeDownloadPrompt();
    }
    // Download the raw DNG image if available
    async function downloadRawImage(img) {
        // Use the dng_file property provided by the API; fallback to swapping extension
        const rawFilename = img.dng_file ?? img.filename.replace(/\.[^/.]+$/,".dng");
        const a = document.createElement("a");
        a.href = `/camapi/view_image/${rawFilename}`;
        a.download = rawFilename;
        a.click();
        closeMenu();
    }
    // Open the image in a new tab for viewing
    function viewImage(img) {
        window.open(img.url, "_blank");
        closeMenu();
    }
    async function deleteImage(img) {
        if (confirm("Are you sure you want to delete this image?")) {
            try {
                await fetch(`/camapi/delete_image/${img.filename}`, {
                    method: "DELETE",
                });
                await loadPage(page);
            } catch (e) {
                console.error("Delete failed", e);
            }
        }
        closeMenu();
    }
</script>

<!-- Add top padding to avoid overlapping header and ensure full viewport height -->
<main class="pt-20 p-4 w-full min-h-screen flex flex-col flex-1">
    <!-- Image grid wrapper to allow scrolling while keeping pagination and back button fixed at bottom -->
    <div class="flex-1 overflow-y-auto mb-4">
        <!-- Responsive grid: more columns on larger screens, tighter gaps for compact layout -->
        <!-- Responsive grid: more columns on larger screens, tighter gaps on mobile -->
        <div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-2 md:gap-4">
            {#each images as img}
                <!-- Container limited to the image size to keep the overlay inside the thumbnail -->
                <div class="relative w-fit h-fit overflow-hidden">
                    <!-- Badge for RAW (DNG) images -->
                    {#if img.has_dng}
                        <span class="absolute top-0 left-0 bg-red-600 text-white text-xs font-bold px-1 py-0.5 rounded-bl">
                            RAW
                        </span>
                    {/if}
                    <!-- Clicking the image opens the action menu -->
                    <img
                        src={img.url}
                        alt=""
                        aria-hidden="true"
                        crossorigin="anonymous"
                        loading="lazy"
                        class="w-full h-auto max-w-xs max-h-48 object-contain rounded border border-white/10"
                        onclick={() => openMenu(img)}
                    />
                    {#if menuOpen === img.filename}
                        <!-- Action menu overlay constrained to the image thumbnail bounds -->
                        <!-- Action menu overlay constrained to the thumbnail bounds -->
                        <!-- Overlay constrained to thumbnail bounds; no scrolling needed -->
                        <div class="absolute inset-0 bg-black/70 flex flex-col items-center justify-center space-y-1 p-1 rounded">
                            <!-- View image button -->
                            <button class="w-full max-w-full px-1 py-0.5 text-xs bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded" onclick={() => viewImage(img)}>
                                View
                            </button>
                            <button class="w-full max-w-full px-1 py-0.5 text-xs bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded" onclick={() => handleDownloadClick(img)}>
                                Download
                            </button>
                            <button class="w-full max-w-full px-1 py-0.5 text-xs bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded" onclick={() => deleteImage(img)}>
                                Delete
                            </button>
                            <button class="w-full max-w-full px-1 py-0.5 text-xs bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded" onclick={closeMenu}>Close</button>
                        </div>
                    {/if}
                </div>
            {/each}
        </div>
    </div>
    <!-- Pagination controls -->
    <!-- Pagination and close button container -->
    <div class="flex flex-col items-center space-y-2 mt-6">
        <div class="flex justify-center space-x-4">
            <button
                onclick={prev}
                class="px-4 py-2 bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded"
                disabled={page <= 1}>Prev</button>
            <span class="self-center text-red-600">Page {page} of {totalPages}</span>
            <button
                onclick={next}
                class="px-4 py-2 bg-black/40 glass border border-white/10 text-red-600 disabled:opacity-50 rounded"
                disabled={page >= totalPages}>Next</button>
        </div>
        <!-- Bottom close button with clearer styling -->
        <button
            class="w-32 py-2 text-red-600 bg-black/60 glass border border-white/10 rounded hover:bg-black/70 transition-colors"
            onclick={() => history.back()}
            aria-label="Go back">✕ Close</button>
    </div>
</main>
    {#if downloadPrompt.open}
        <!-- Simple modal for download choice -->
        <div class="fixed inset-0 bg-black/70 flex items-center justify-center z-50">
            <div class="bg-gray-800 p-4 rounded shadow-lg max-w-xs w-full">
                <p class="text-white mb-2">Download which format?</p>
                <div class="flex justify-between space-x-2">
                    <button class="flex-1 px-2 py-1 bg-black/40 glass border border-white/10 text-red-600 rounded" onclick={confirmDownloadJpg}>JPG</button>
                    <button class="flex-1 px-2 py-1 bg-black/40 glass border border-white/10 text-red-600 rounded" onclick={confirmDownloadRaw}>RAW</button>
                </div>
                <button class="w-full mt-2 px-2 py-1 bg-black/40 glass border border-white/10 text-red-600 rounded" onclick={closeDownloadPrompt}>Cancel</button>
            </div>
        </div>
    {/if}
