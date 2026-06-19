<script>
import { coilToTikz } from '/WebSharedComponents/assets/js/coilToTikz.js';
import { MOCK_COILS } from '/WebSharedComponents/assets/js/mockCoils.js';

// Proof-of-concept playground: pick (or paste) a coil, generate its electrical
// schematic as TikZ via coilToTikz(), and render it to a PDF through the existing
// /process_latex backend endpoint.
export default {
    name: 'SchematicPlayground',
    data() {
        const base = import.meta.env.VITE_API_ENDPOINT;
        return {
            mocks: MOCK_COILS,
            selected: 0,
            coilText: '',
            tikz: '',
            pdf: null,
            // Production serves /process_latex off VITE_API_ENDPOINT. For local dev,
            // default to the OM WebBackend on :8001 (the canonical :8000 is taken by
            // the prebuilt reference container here). Field is editable.
            endpoint: base ? `${base}/process_latex` : 'http://localhost:8001/process_latex',
            error: '',
            rendering: false,
        };
    },
    mounted() {
        this.loadMock(0);
    },
    methods: {
        loadMock(index) {
            this.selected = index;
            this.coilText = JSON.stringify(this.mocks[index].coil, null, 2);
            this.generate();
        },
        generate() {
            this.error = '';
            this.pdf = null;
            try {
                const coil = JSON.parse(this.coilText);
                this.tikz = coilToTikz(coil);
            } catch (e) {
                this.tikz = '';
                this.error = `Could not generate TikZ: ${e.message}`;
            }
        },
        renderPdf() {
            if (!this.tikz) return;
            this.rendering = true;
            this.error = '';
            this.$axios.post(this.endpoint, this.tikz)
                .then((response) => {
                    this.pdf = `data:application/pdf;base64,${response.data}`;
                })
                .catch((err) => {
                    this.error = `Render failed (${this.endpoint}): ${err.message}. `
                        + 'The TikZ above is still valid -- point the endpoint at a '
                        + 'backend that exposes /process_latex.';
                })
                .finally(() => { this.rendering = false; });
        },
    },
};
</script>

<template>
    <div class="playground">
        <div class="left">
            <h2>Schematic Playground</h2>
            <p class="hint">
                Pick a mock coil (or edit the JSON), generate its TikZ schematic, then
                render it to PDF via <code>/process_latex</code>.
            </p>

            <label class="lbl">Mock coil</label>
            <div class="mocks">
                <button v-for="(m, i) in mocks" :key="m.name"
                        :class="['chip', { active: i === selected }]"
                        @click="loadMock(i)">{{ m.name }}</button>
            </div>
            <p class="caption">{{ mocks[selected].caption }}</p>

            <label class="lbl">Coil JSON (editable)</label>
            <textarea v-model="coilText" class="json" spellcheck="false"></textarea>

            <label class="lbl">Render endpoint</label>
            <input v-model="endpoint" class="endpoint" spellcheck="false" />

            <div class="actions">
                <button class="btn" @click="generate">Generate TikZ</button>
                <button class="btn primary" :disabled="rendering || !tikz" @click="renderPdf">
                    {{ rendering ? 'Rendering...' : 'Render PDF' }}
                </button>
            </div>

            <p v-if="error" class="error">{{ error }}</p>

            <label class="lbl">Generated TikZ</label>
            <pre class="tikz">{{ tikz }}</pre>
        </div>

        <div class="right">
            <iframe v-if="pdf" :src="pdf" class="pdf" title="schematic"></iframe>
            <div v-else class="placeholder">
                <p>Rendered schematic appears here.</p>
                <p class="hint">Click <strong>Render PDF</strong> to compile the TikZ.</p>
            </div>
        </div>
    </div>
</template>

<style scoped>
/* OpenMagnetics palette (mirrors the --bs-* tokens in assets/css/custom.css).
   Hard-coded because those vars are only set after the engine-loader init, which
   this route skips: teal #539796, dark #1a1a1a, panel #2a2a2a, text #d4d4d4. */
.playground { display: flex; gap: 1rem; padding: 1rem; height: 100vh; box-sizing: border-box;
    color: #d4d4d4; background: #1a1a1a; }
.left { width: 480px; min-width: 380px; display: flex; flex-direction: column; overflow-y: auto;
    border: 1px solid #539796; border-radius: 6px; padding: 1rem; }
.right { flex: 1; min-width: 300px; border: 1px solid #539796; border-radius: 6px; padding: .5rem; }
h2 { margin: 0 0 .25rem; color: #539796; font-size: 1.5rem; }
.hint { font-size: .85rem; opacity: .7; margin: .25rem 0; }
.hint code { color: #539796; }
.lbl { font-size: .8rem; text-transform: uppercase; letter-spacing: .05em; opacity: .6; margin-top: .75rem; }
.mocks { display: flex; flex-wrap: wrap; gap: .35rem; margin-top: .35rem; }
.chip { padding: .25rem .6rem; border: 1px solid #539796; border-radius: 999px;
    background: transparent; color: #d4d4d4; cursor: pointer; font-size: .8rem; }
.chip.active { background: #539796; color: #1a1a1a; }
.caption { font-size: .85rem; color: #539796; opacity: .9; margin: .4rem 0 0; }
.json, .endpoint { font-family: monospace; font-size: .8rem; margin-top: .35rem; padding: .35rem;
    background: #2a2a2a; color: #d4d4d4; border: 1px solid rgba(83, 151, 150, .45); border-radius: 4px; }
.json { height: 200px; resize: vertical; }
.actions { display: flex; gap: .5rem; margin-top: .75rem; }
.btn { padding: .45rem .9rem; border: 1px solid #539796; border-radius: 6px;
    background: transparent; color: #539796; cursor: pointer; }
.btn.primary { background: #539796; color: #1a1a1a; }
.btn:disabled { opacity: .5; cursor: default; }
.error { color: #ff8800; font-size: .85rem; margin-top: .5rem; }
.tikz { flex: 1; min-height: 160px; background: #000; color: #d4d4d4; padding: .5rem;
    font-size: .72rem; overflow: auto; white-space: pre; border-radius: 6px;
    border: 1px solid rgba(83, 151, 150, .35); }
.pdf { width: 100%; height: 100%; border: none; background: #fff; border-radius: 4px; }
.placeholder { display: flex; flex-direction: column; align-items: center; justify-content: center;
    height: 100%; opacity: .6; border: 1px dashed rgba(83, 151, 150, .5); border-radius: 6px; }
</style>
