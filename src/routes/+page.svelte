<script>
    import { onMount } from "svelte";
    // Capture a still image from the camera (POST request)
    async function capture() {
        try {
            await fetch("/camapi/capture_still_0", { method: "POST" });
        } catch (e) {
            console.error("Capture failed", e);
        }
    }
    // --- ISO control logic ---
    // Define typical ISO steps (similar to mobile camera apps)
    const isoSteps = [100, 200, 400, 800, 1600, 3200, 6400];
    let isoIndex = $state(0);
    let isoValue = $state(isoSteps[isoIndex]);

    // Save RAW flag (boolean) – will be initialized from camera profile on load
    let saveRAW = $state(false);

    // Generic helper to send any setting update to the backend
    async function updateSetting({ camera_num = 0, id, value }) {
        try {
            const res = await fetch("/camapi/update_setting", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ camera_num, id, value }),
            });
            if (!res.ok) {
                const err = await res.text();
                console.error("Update setting failed:", err);
            }
        } catch (e) {
            console.error("Update setting error:", e);
        }
    }

    // ISO specific helper that uses the generic updateSetting
    function setISO(idx) {
        isoIndex = idx;
        isoValue = isoSteps[isoIndex];
        // Convert ISO to AnalogueGain (picamera2 expects a float gain, typically ISO/100)
        const gain = isoValue / 100;
        // Send the gain to the backend using the proper control name
        updateSetting({ id: "AnalogueGain", value: gain });
    }

    // --- Exposure control logic ---
    // Define typical exposure times (seconds) similar to mobile apps
    const exposureSteps = [
        1 / 8000,
        1 / 4000,
        1 / 2000,
        1 / 1000,
        1 / 500,
        1 / 250,
        1 / 125,
        1 / 60,
        1 / 30,
        1 / 15,
        1 / 8,
        1 / 4,
        1 / 2,
        1,
        2,
        3,
        4,
        5,
        6,
        8,
        10,
        15,
        20,
        30
    ];
    let exposureIndex = $state(0);
    let exposureValue = $state(exposureSteps[exposureIndex]); // seconds

    // Helper to format exposure for display (e.g., 1/125 or 1s)
    function formatExposure(val) {
        if (val >= 1) return `${val}s`;
        const denom = Math.round(1 / val);
        return `1/${denom}`;
    }

    // Helper to set exposure (convert seconds to microseconds for picamera2)
    function setExposure(idx) {
        exposureIndex = idx;
        exposureValue = exposureSteps[exposureIndex];
        const microseconds = Math.round(exposureValue * 1_000_000);
        updateSetting({ id: "ExposureTime", value: microseconds });
    }

    // Toggle Save RAW flag and persist to backend
    function toggleSaveRAW() {
        saveRAW = !saveRAW;
        updateSetting({ id: "saveRAW", value: saveRAW });
    }

    // Store fetched metadata for debugging (will be logged to console)
    let metadata = $state({});
    // Store fetched camera profile for Save RAW flag
    let cameraProfile = $state(null);

    // Load initial data: fetch camera profile for defaults and metadata for logging only
    onMount(async () => {
        try {
            // Fetch profile (used for ISO, exposure, Save RAW defaults)
            const profileRes = await fetch(
                "/camapi/get_camera_profile?camera_num=0",
            );
            const profileData = await profileRes.json();
            cameraProfile = profileData;
            console.log("Camera profile loaded:", profileData);

            // --- ISO initialization from profile ---
            const gain =
                profileData?.camera_profile?.controls?.AnalogueGain ??
                profileData?.controls?.AnalogueGain;
            if (gain !== undefined) {
                const iso = Math.round(gain * 100);
                let idx = isoSteps.findIndex((step) => step >= iso);
                if (idx === -1) idx = isoSteps.length - 1;
                isoIndex = idx;
                isoValue = isoSteps[isoIndex];
            }
            // --- Exposure initialization from profile ---
            const expMicro =
                profileData?.camera_profile?.controls?.ExposureTime ??
                profileData?.controls?.ExposureTime;
            if (expMicro !== undefined) {
                const expSec = expMicro / 1_000_000;
                let idx = exposureSteps.findIndex((step) => step >= expSec);
                if (idx === -1) idx = exposureSteps.length - 1;
                exposureIndex = idx;
                exposureValue = exposureSteps[exposureIndex];
            }
            // --- Save RAW initialization from profile ---
            const rawFlag =
                profileData?.camera_profile?.saveRAW ?? profileData?.saveRAW;
            if (rawFlag !== undefined) {
                saveRAW = rawFlag;
            }

            // Fetch metadata only for logging purposes
            const metaRes = await fetch("/camapi/fetch_metadata_0");
            const metaData = await metaRes.json();
            metadata = metaData;
            console.log("Metadata loaded:", metaData);
        } catch (e) {
            console.error("Failed to load initial data", e);
        }
    });
</script>

<!-- Restored a centered layout with a max width for wide screens -->
<main class="pt-20 pb-32 px-4 lg:px-8 max-w-screen-lg mx-auto space-y-6">
    <!-- Sensor Feed Preview – click to open fullscreen view -->
    <a href="/camera" class="block">
        <section
            class="relative w-full aspect-video rounded-lg overflow-hidden border border-white/10 bg-black group"
        >
            <div class="absolute inset-0 z-0">
                <img
                    alt="Live Sensor Feed"
                    class="w-full h-full object-cover opacity-100 group-hover:opacity-80 transition-opacity duration-700"
                    src="camapi/video_feed_0"
                />
            </div>
            <!-- Overlay Info -->
            <div
                class="absolute top-2 left-2 flex items-center gap-2 bg-black/60 glass px-2 py-1 rounded border border-white/10"
            >
                <div
                    class="w-2 h-2 rounded-full bg-red-600 animate-pulse"
                ></div>
                <span
                    class="font-mono text-[10px] text-red-600 uppercase tracking-widest"
                    >LIVE SENSOR</span
                >
            </div>
            <!-- Focus Crosshair -->
            <div
                class="absolute inset-0 flex items-center justify-center pointer-events-none"
            >
                <div class="absolute h-px w-4 bg-red-600"></div>
                <div class="absolute w-px h-4 bg-red-600"></div>
            </div>
        </section>
    </a>
    <!-- Capture button just under live preview – full width -->
    <div class="mt-4">
        <button
            class="relative group active:scale-90 transition-transform duration-150 outline-none w-full py-2 bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
            onclick={capture}
        >
            <div
                class="absolute inset-0 rounded-full border-2 border-primary-container/30 -m-2"
            ></div>
            <span class="material-symbols-outlined text-4xl mx-auto block"
                >photo_camera</span
            >
        </button>
    </div>
    <!-- Gallery button – full width, same style -->
    <div class="mt-2">
        <a
            href="/gallery"
            class="block w-full py-2 bg-black/40 glass border border-white/10 text-red-600 text-center hover:bg-black/60 transition-colors rounded"
        >
            GALLERY
        </a>
    </div>
    <!-- Manual Controls Bento -->
    <div class="grid grid-cols-1 gap-panel-gap">
        <!-- ISO and Shutter Control Card -->
        <div
            class="p-2 rounded-xl space-y-6"
        >
            <div class="flex flex-col space-y-4">
                <div class="flex items-center justify-between">
                    <span
                        class="font-label-caps text-label-caps text-zinc-500 uppercase"
                        >ISO SENSITIVITY</span
                    >
                </div>
                <div
                    class="flex items-center justify-between bg-surface-container-low rounded-lg p-1 border border-white/5"
                >
                    <!-- ISO decrement button -->
                    <button
                        class="w-12 h-12 flex items-center justify-center bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
                        onclick={() => setISO(Math.max(0, isoIndex - 1))}
                    >
                        <span
                            class="material-symbols-outlined"
                            data-icon="remove">remove</span
                        >
                    </button>
                    <div class="flex flex-col items-center">
                        <!-- Display current ISO value -->
                        <span
                            class="font-numeral-xl text-numeral-xl text-on-surface"
                            >{isoValue}</span
                        >
                        <!-- Optional placeholder for gain info (could be calculated later) -->
                        <span
                            class="text-[8px] font-mono text-zinc-500 tracking-[0.2em]"
                            >ISO</span
                        >
                    </div>
                    <!-- ISO increment button -->
                    <button
                        class="w-12 h-12 flex items-center justify-center bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
                        onclick={() =>
                            setISO(Math.min(isoSteps.length - 1, isoIndex + 1))}
                    >
                        <span class="material-symbols-outlined" data-icon="add"
                            >add</span
                        >
                    </button>
                </div>
            </div>
            <div class="flex flex-col space-y-4">
                <div class="flex items-center justify-between">
                    <span
                        class="font-label-caps text-label-caps text-zinc-500 uppercase"
                        >SHUTTER SPEED</span
                    >
                </div>
                <div
                    class="flex items-center justify-between bg-surface-container-low rounded-lg p-1 border border-white/5"
                >
                    <button
                        class="w-12 h-12 flex items-center justify-center bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
                        onclick={() =>
                            setExposure(Math.max(0, exposureIndex - 1))}
                    >
                        <span
                            class="material-symbols-outlined"
                            data-icon="remove">remove</span
                        >
                    </button>
                    <div class="flex flex-col items-center">
                        <span
                            class="font-numeral-xl text-numeral-xl text-on-surface"
                            >{formatExposure(exposureValue)}</span
                        >
                        <span
                            class="text-[8px] font-mono text-zinc-500 tracking-[0.2em]"
                            >EXPOSURE</span
                        >
                    </div>
                    <button
                        class="w-12 h-12 flex items-center justify-center bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
                        onclick={() =>
                            setExposure(
                                Math.min(
                                    exposureSteps.length - 1,
                                    exposureIndex + 1,
                                ),
                            )}
                    >
                        <span class="material-symbols-outlined" data-icon="add"
                            >add</span
                        >
                    </button>
                </div>
            </div>
        </div>
        <!-- Image Format -->
        <div class="grid gap-panel-gap">
            <div class="flex items-center justify-between mt-4">
                <span
                    class="font-label-caps text-label-caps text-zinc-500 uppercase"
                    >SAVE RAW</span
                >
                <button
                    class="w-12 h-12 flex items-center justify-center bg-black/40 hover:bg-red-900/20 active:scale-95 transition-all text-red-600 rounded"
                    onclick={toggleSaveRAW}
                >
                    {#if saveRAW}
                        <span class="material-symbols-outlined"
                            >check_circle</span
                        >
                    {:else}
                        <span class="material-symbols-outlined">circle</span>
                    {/if}
                </button>
            </div>
        </div>
    </div>
</main>
