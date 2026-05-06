<main class="fixed inset-0 z-20 flex flex-col bg-black pt-16">
    <!-- Live feed image fills remaining space but constrained to avoid overflow -->
    <div class="flex-1 flex items-center justify-center overflow-hidden">
        <img
            src="camapi/video_feed_0"
            alt="Live Sensor Feed"
            class="max-w-full max-h-full object-contain"
        />
    </div>
    <!-- Bottom back button spanning full width -->
    <!-- Consistent full‑width back button matching red/black theme -->
    <button
        class="w-full py-3 text-red-600 bg-black/60 glass border-t border-white/10 hover:bg-black/70 transition-colors z-10"
        onclick={() => history.back()}
        aria-label="Go back">✕ Close</button
    >
</main>
