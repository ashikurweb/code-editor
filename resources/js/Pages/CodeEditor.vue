<script setup>
import { ref, computed, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';
import { Head } from '@inertiajs/vue3';

// Monaco Editor imports
import * as monaco from 'monaco-editor';

// Configure Monaco workers
self.MonacoEnvironment = {
    getWorker(_, label) {
        const getWorkerModule = (url, options) => {
            return new Worker(url, { type: 'module', ...options });
        };
        if (label === 'css' || label === 'scss' || label === 'less') {
            return getWorkerModule(
                new URL('monaco-editor/esm/vs/language/css/css.worker.js', import.meta.url)
            );
        }
        if (label === 'html' || label === 'handlebars' || label === 'razor') {
            return getWorkerModule(
                new URL('monaco-editor/esm/vs/language/html/html.worker.js', import.meta.url)
            );
        }
        if (label === 'javascript' || label === 'typescript') {
            return getWorkerModule(
                new URL('monaco-editor/esm/vs/language/typescript/ts.worker.js', import.meta.url)
            );
        }
        return getWorkerModule(
            new URL('monaco-editor/esm/vs/editor/editor.worker.js', import.meta.url)
        );
    },
};

// Reactive state
const activeTab = ref('html');
const isPreviewMaximized = ref(false);
const isEditorMaximized = ref(false);
const previewRef = ref(null);
const editorContainerRef = ref(null);
const showConsole = ref(false);
const consoleLogs = ref([]);
const layout = ref('horizontal'); // 'horizontal' or 'vertical'
const fontSize = ref(14);
const wordWrap = ref('off');

// Editor instances
let htmlEditor = null;
let cssEditor = null;
let jsEditor = null;
let currentEditor = null;
let debounceTimer = null;

// Default code templates
const defaultHTML = `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Preview</title>
</head>
<body>
    <div class="container">
        <h1>Hello, World! 🚀</h1>
        <p>Start editing to see your changes live.</p>
        <button id="actionBtn" onclick="handleClick()">Click Me</button>
        <div id="output"></div>
    </div>
</body>
</html>`;

const defaultCSS = `* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
    color: #e2e8f0;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
}

.container {
    text-align: center;
    padding: 3rem;
    background: rgba(255, 255, 255, 0.05);
    border-radius: 1.5rem;
    backdrop-filter: blur(20px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5);
    max-width: 600px;
    width: 90%;
    animation: fadeInUp 0.8s ease-out;
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(30px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

h1 {
    font-size: 2.5rem;
    font-weight: 800;
    background: linear-gradient(135deg, #667eea, #764ba2, #f093fb);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    margin-bottom: 1rem;
}

p {
    color: #a0aec0;
    font-size: 1.1rem;
    margin-bottom: 2rem;
    line-height: 1.6;
}

button {
    padding: 0.8rem 2.5rem;
    font-size: 1rem;
    font-weight: 600;
    color: #fff;
    background: linear-gradient(135deg, #667eea, #764ba2);
    border: none;
    border-radius: 0.75rem;
    cursor: pointer;
    transition: all 0.3s ease;
    letter-spacing: 0.025em;
}

button:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 30px rgba(102, 126, 234, 0.4);
}

button:active {
    transform: translateY(0);
}

#output {
    margin-top: 1.5rem;
    padding: 1rem;
    font-size: 1rem;
    color: #68d391;
    min-height: 1.5rem;
    transition: all 0.3s ease;
}`;

const defaultJS = `let clickCount = 0;

function handleClick() {
    clickCount++;
    const output = document.getElementById('output');
    const messages = [
        '🎉 Nice click!',
        '⚡ You\\'re on fire!',
        '🚀 Keep going!',
        '✨ Amazing!',
        '🎯 Bull\\'s eye!',
        '💎 Brilliant!',
        '🌟 Superstar!',
        '🔥 Unstoppable!'
    ];
    const msg = messages[clickCount % messages.length];
    output.innerHTML = \`\${msg} (clicked \${clickCount} time\${clickCount > 1 ? 's' : ''})\`;
    output.style.animation = 'none';
    output.offsetHeight; // trigger reflow
    output.style.animation = 'fadeInUp 0.4s ease-out';
    console.log(\`Button clicked \${clickCount} time(s)\`);
}

console.log('🚀 Code Editor loaded successfully!');
console.log('Try clicking the button!');`;

// Tab definitions
const tabs = [
    { id: 'html', label: 'HTML', icon: '🌐', color: '#e34c26' },
    { id: 'css', label: 'CSS', icon: '🎨', color: '#264de4' },
    { id: 'javascript', label: 'JavaScript', icon: '⚡', color: '#f7df1e' },
];

// Editor theme definition
const defineEditorTheme = () => {
    monaco.editor.defineTheme('codeEditorDark', {
        base: 'vs-dark',
        inherit: true,
        rules: [
            { token: 'comment', foreground: '6a737d', fontStyle: 'italic' },
            { token: 'keyword', foreground: 'c792ea' },
            { token: 'string', foreground: 'c3e88d' },
            { token: 'number', foreground: 'f78c6c' },
            { token: 'tag', foreground: 'f07178' },
            { token: 'attribute.name', foreground: 'ffcb6b' },
            { token: 'attribute.value', foreground: 'c3e88d' },
            { token: 'delimiter', foreground: '89ddff' },
            { token: 'type', foreground: '82aaff' },
            { token: 'variable', foreground: 'eeffff' },
            { token: 'operator', foreground: '89ddff' },
        ],
        colors: {
            'editor.background': '#0d1117',
            'editor.foreground': '#c9d1d9',
            'editor.lineHighlightBackground': '#161b22',
            'editor.selectionBackground': '#264f78',
            'editorCursor.foreground': '#79c0ff',
            'editor.inactiveSelectionBackground': '#1c2a3a',
            'editorLineNumber.foreground': '#484f58',
            'editorLineNumber.activeForeground': '#79c0ff',
            'editorIndentGuide.background': '#21262d',
            'editorIndentGuide.activeBackground': '#30363d',
            'editor.selectionHighlightBackground': '#1c3a5f55',
            'editorBracketMatch.background': '#1c3a5f55',
            'editorBracketMatch.border': '#79c0ff44',
            'scrollbar.shadow': '#0008',
            'scrollbarSlider.background': '#484f5833',
            'scrollbarSlider.hoverBackground': '#484f5855',
            'scrollbarSlider.activeBackground': '#484f5877',
            'minimap.background': '#0d1117',
        },
    });
};

// Create editor options
const getEditorOptions = (language) => ({
    language,
    theme: 'codeEditorDark',
    fontSize: fontSize.value,
    fontFamily: "'JetBrains Mono', 'Fira Code', 'Cascadia Code', monospace",
    fontLigatures: true,
    lineNumbers: 'on',
    minimap: { enabled: true, scale: 1, renderCharacters: false },
    scrollBeyondLastLine: false,
    automaticLayout: true,
    tabSize: 2,
    wordWrap: wordWrap.value,
    formatOnPaste: true,
    formatOnType: false,
    renderWhitespace: 'selection',
    smoothScrolling: true,
    cursorBlinking: 'smooth',
    cursorSmoothCaretAnimation: 'on',
    roundedSelection: true,
    padding: { top: 16, bottom: 16 },
    bracketPairColorization: { enabled: true },
    guides: { bracketPairs: true, indentation: true },
    suggest: { showMethods: true, showFunctions: true, showSnippets: true },
    fixedOverflowWidgets: true,
    scrollbar: {
        verticalScrollbarSize: 8,
        horizontalScrollbarSize: 8,
        useShadows: false,
    },
});

// Initialize editors
const initEditors = () => {
    defineEditorTheme();

    const container = editorContainerRef.value;
    if (!container) return;

    // Create hidden div for each editor
    const htmlDiv = document.createElement('div');
    htmlDiv.id = 'html-editor';
    htmlDiv.style.cssText = 'width:100%;height:100%;';
    container.appendChild(htmlDiv);

    const cssDiv = document.createElement('div');
    cssDiv.id = 'css-editor';
    cssDiv.style.cssText = 'width:100%;height:100%;display:none;';
    container.appendChild(cssDiv);

    const jsDiv = document.createElement('div');
    jsDiv.id = 'js-editor';
    jsDiv.style.cssText = 'width:100%;height:100%;display:none;';
    container.appendChild(jsDiv);

    htmlEditor = monaco.editor.create(htmlDiv, {
        ...getEditorOptions('html'),
        value: defaultHTML,
    });

    cssEditor = monaco.editor.create(cssDiv, {
        ...getEditorOptions('css'),
        value: defaultCSS,
    });

    jsEditor = monaco.editor.create(jsDiv, {
        ...getEditorOptions('javascript'),
        value: defaultJS,
    });

    currentEditor = htmlEditor;

    // Listen for changes
    [htmlEditor, cssEditor, jsEditor].forEach((editor) => {
        editor.onDidChangeModelContent(() => {
            debouncedUpdatePreview();
        });
    });

    // Initial preview
    updatePreview();
};

// Switch active tab
const switchTab = (tabId) => {
    activeTab.value = tabId;

    const htmlDiv = document.getElementById('html-editor');
    const cssDiv = document.getElementById('css-editor');
    const jsDiv = document.getElementById('js-editor');

    htmlDiv.style.display = tabId === 'html' ? 'block' : 'none';
    cssDiv.style.display = tabId === 'css' ? 'block' : 'none';
    jsDiv.style.display = tabId === 'javascript' ? 'block' : 'none';

    if (tabId === 'html') currentEditor = htmlEditor;
    else if (tabId === 'css') currentEditor = cssEditor;
    else currentEditor = jsEditor;

    nextTick(() => {
        currentEditor?.layout();
        currentEditor?.focus();
    });
};

// Debounced preview update (500ms)
const debouncedUpdatePreview = () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(updatePreview, 500);
};

// Update the live preview
const updatePreview = () => {
    if (!previewRef.value) return;

    const html = htmlEditor?.getValue() || '';
    const css = cssEditor?.getValue() || '';
    const js = jsEditor?.getValue() || '';

    const previewContent = `
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <style>${css}</style>
        </head>
        <body>
            ${extractBodyContent(html)}
            <script>
                // Console override to capture logs
                (function() {
                    const originalLog = console.log;
                    const originalError = console.error;
                    const originalWarn = console.warn;
                    const originalInfo = console.info;

                    function sendToParent(type, args) {
                        try {
                            window.parent.postMessage({
                                type: 'console',
                                method: type,
                                args: Array.from(args).map(a => {
                                    try { return typeof a === 'object' ? JSON.stringify(a) : String(a); }
                                    catch(e) { return String(a); }
                                })
                            }, '*');
                        } catch(e) {}
                    }

                    console.log = function() { sendToParent('log', arguments); originalLog.apply(console, arguments); };
                    console.error = function() { sendToParent('error', arguments); originalError.apply(console, arguments); };
                    console.warn = function() { sendToParent('warn', arguments); originalWarn.apply(console, arguments); };
                    console.info = function() { sendToParent('info', arguments); originalInfo.apply(console, arguments); };

                    window.onerror = function(msg, url, lineNo, colNo, error) {
                        sendToParent('error', [msg + ' (Line: ' + lineNo + ')']);
                        return false;
                    };
                })();

                try {
                    ${js}
                } catch(e) {
                    console.error(e.message);
                }
            <\/script>
        </body>
        </html>
    `;

    const iframe = previewRef.value;
    const doc = iframe.contentDocument || iframe.contentWindow.document;
    doc.open();
    doc.write(previewContent);
    doc.close();
};

// Extract body content from full HTML
const extractBodyContent = (html) => {
    const bodyMatch = html.match(/<body[^>]*>([\s\S]*?)<\/body>/i);
    if (bodyMatch) return bodyMatch[1];

    // If no body tag, check if it has head/html tags and extract after them
    const headEnd = html.indexOf('</head>');
    if (headEnd !== -1) {
        const afterHead = html.substring(headEnd + 7);
        const bodyContent = afterHead.replace(/<\/?html[^>]*>/gi, '').replace(/<\/?body[^>]*>/gi, '');
        return bodyContent;
    }

    return html;
};

// Console message handler
const handleConsoleMessage = (event) => {
    if (event.data && event.data.type === 'console') {
        consoleLogs.value.push({
            method: event.data.method,
            args: event.data.args,
            timestamp: new Date().toLocaleTimeString(),
        });
        // Keep only last 100 logs
        if (consoleLogs.value.length > 100) {
            consoleLogs.value = consoleLogs.value.slice(-100);
        }
    }
};

// Format code
const formatCode = () => {
    currentEditor?.getAction('editor.action.formatDocument')?.run();
};

// Toggle layout
const toggleLayout = () => {
    layout.value = layout.value === 'horizontal' ? 'vertical' : 'horizontal';
    nextTick(() => {
        htmlEditor?.layout();
        cssEditor?.layout();
        jsEditor?.layout();
    });
};

// Clear console
const clearConsole = () => {
    consoleLogs.value = [];
};

// Reset editors to default
const resetEditors = () => {
    htmlEditor?.setValue(defaultHTML);
    cssEditor?.setValue(defaultCSS);
    jsEditor?.setValue(defaultJS);
    consoleLogs.value = [];
    updatePreview();
};

// Increase font size
const increaseFontSize = () => {
    fontSize.value = Math.min(fontSize.value + 1, 24);
    [htmlEditor, cssEditor, jsEditor].forEach(e => e?.updateOptions({ fontSize: fontSize.value }));
};

// Decrease font size
const decreaseFontSize = () => {
    fontSize.value = Math.max(fontSize.value - 1, 10);
    [htmlEditor, cssEditor, jsEditor].forEach(e => e?.updateOptions({ fontSize: fontSize.value }));
};

// Toggle word wrap
const toggleWordWrap = () => {
    wordWrap.value = wordWrap.value === 'off' ? 'on' : 'off';
    [htmlEditor, cssEditor, jsEditor].forEach(e => e?.updateOptions({ wordWrap: wordWrap.value }));
};

// Download code as HTML file
const downloadCode = () => {
    const html = htmlEditor?.getValue() || '';
    const css = cssEditor?.getValue() || '';
    const js = jsEditor?.getValue() || '';

    const fullHTML = html.replace('</head>', `<style>\n${css}\n</style>\n</head>`).replace('</body>', `<script>\n${js}\n<\/script>\n</body>`);

    const blob = new Blob([fullHTML], { type: 'text/html' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'code-editor-export.html';
    a.click();
    URL.revokeObjectURL(url);
};

// Console log type icon/color
const consoleIcon = (method) => {
    switch (method) {
        case 'error': return '❌';
        case 'warn': return '⚠️';
        case 'info': return 'ℹ️';
        default: return '›';
    }
};

const consoleColor = (method) => {
    switch (method) {
        case 'error': return 'text-red-400';
        case 'warn': return 'text-yellow-400';
        case 'info': return 'text-blue-400';
        default: return 'text-emerald-400';
    }
};

// Lifecycle
onMounted(() => {
    nextTick(() => initEditors());
    window.addEventListener('message', handleConsoleMessage);
});

onBeforeUnmount(() => {
    htmlEditor?.dispose();
    cssEditor?.dispose();
    jsEditor?.dispose();
    clearTimeout(debounceTimer);
    window.removeEventListener('message', handleConsoleMessage);
});
</script>

<template>
    <Head title="Code Editor" />

    <div class="h-screen w-screen flex flex-col overflow-hidden bg-[#010409]">
        <!-- Top Header Bar -->
        <header class="flex items-center justify-between px-4 h-12 bg-[#0d1117] border-b border-[#21262d] shrink-0 z-50">
            <!-- Logo & Title -->
            <div class="flex items-center gap-3">
                <div class="flex items-center gap-2">
                    <div class="w-7 h-7 rounded-lg bg-gradient-to-br from-violet-500 to-fuchsia-500 flex items-center justify-center shadow-lg shadow-violet-500/20">
                        <svg class="w-4 h-4 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 20l4-16m4 4l4 4-4 4M6 16l-4-4 4-4" />
                        </svg>
                    </div>
                    <span class="text-sm font-bold text-white tracking-tight">Code<span class="text-violet-400">Editor</span></span>
                </div>
                <div class="hidden sm:block h-4 w-px bg-[#30363d]"></div>
                <span class="hidden sm:block text-xs text-[#8b949e] font-medium">Live Preview</span>
            </div>

            <!-- Toolbar Actions -->
            <div class="flex items-center gap-1">
                <!-- Font Size Controls -->
                <div class="hidden md:flex items-center gap-0.5 mr-2 bg-[#161b22] rounded-lg px-1 py-0.5 border border-[#21262d]">
                    <button @click="decreaseFontSize" class="p-1.5 text-[#8b949e] hover:text-white transition-colors rounded" title="Decrease font size">
                        <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20 12H4"/></svg>
                    </button>
                    <span class="text-xs text-[#8b949e] font-mono px-1 min-w-[2rem] text-center">{{ fontSize }}</span>
                    <button @click="increaseFontSize" class="p-1.5 text-[#8b949e] hover:text-white transition-colors rounded" title="Increase font size">
                        <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4"/></svg>
                    </button>
                </div>

                <!-- Word Wrap Toggle -->
                <button @click="toggleWordWrap" class="p-1.5 rounded-lg transition-all duration-200"
                    :class="wordWrap === 'on' ? 'bg-violet-500/20 text-violet-400' : 'text-[#8b949e] hover:text-white hover:bg-[#161b22]'"
                    title="Toggle word wrap">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 10h10a5 5 0 010 10H9m0 0l3-3m-3 3l3 3M3 6h18"/></svg>
                </button>

                <!-- Format Code -->
                <button @click="formatCode" class="p-1.5 text-[#8b949e] hover:text-white hover:bg-[#161b22] rounded-lg transition-all duration-200" title="Format code">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h8m-8 6h16"/></svg>
                </button>

                <!-- Layout Toggle -->
                <button @click="toggleLayout" class="p-1.5 text-[#8b949e] hover:text-white hover:bg-[#161b22] rounded-lg transition-all duration-200" title="Toggle layout">
                    <svg v-if="layout === 'horizontal'" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 4v16M4 4h16v16H4z"/></svg>
                    <svg v-else class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 12h16M4 4h16v16H4z"/></svg>
                </button>

                <!-- Console Toggle -->
                <button @click="showConsole = !showConsole" class="p-1.5 rounded-lg transition-all duration-200 relative"
                    :class="showConsole ? 'bg-violet-500/20 text-violet-400' : 'text-[#8b949e] hover:text-white hover:bg-[#161b22]'"
                    title="Toggle console">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 9l3 3-3 3m5 0h3M5 20h14a2 2 0 002-2V6a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/></svg>
                    <span v-if="consoleLogs.length > 0" class="absolute -top-0.5 -right-0.5 w-3.5 h-3.5 bg-violet-500 rounded-full text-[8px] text-white font-bold flex items-center justify-center">
                        {{ consoleLogs.length > 9 ? '9+' : consoleLogs.length }}
                    </span>
                </button>

                <div class="h-4 w-px bg-[#30363d] mx-1"></div>

                <!-- Reset -->
                <button @click="resetEditors" class="p-1.5 text-[#8b949e] hover:text-red-400 hover:bg-red-500/10 rounded-lg transition-all duration-200" title="Reset to default">
                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/></svg>
                </button>

                <!-- Download -->
                <button @click="downloadCode" class="flex items-center gap-1.5 px-3 py-1.5 bg-gradient-to-r from-violet-600 to-fuchsia-600 hover:from-violet-500 hover:to-fuchsia-500 text-white text-xs font-semibold rounded-lg transition-all duration-200 shadow-lg shadow-violet-500/20 hover:shadow-violet-500/30" title="Download HTML">
                    <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"/></svg>
                    <span class="hidden sm:inline">Export</span>
                </button>
            </div>
        </header>

        <!-- Main Content Area -->
        <div class="flex-1 flex overflow-hidden" :class="layout === 'vertical' ? 'flex-col' : 'flex-row'">
            <!-- Editor Panel -->
            <div class="flex flex-col overflow-hidden"
                :class="[
                    isEditorMaximized ? 'flex-1' : '',
                    isPreviewMaximized ? 'hidden' : '',
                    !isEditorMaximized && !isPreviewMaximized ? (layout === 'horizontal' ? 'w-1/2' : 'h-1/2') : '',
                    isEditorMaximized ? 'w-full h-full' : ''
                ]"
            >
                <!-- Tab Bar -->
                <div class="flex items-center justify-between bg-[#0d1117] border-b border-[#21262d] shrink-0">
                    <div class="flex items-center">
                        <button
                            v-for="tab in tabs"
                            :key="tab.id"
                            @click="switchTab(tab.id)"
                            class="group relative flex items-center gap-2 px-4 py-2.5 text-xs font-medium transition-all duration-200 border-b-2"
                            :class="activeTab === tab.id
                                ? 'text-white border-violet-500 bg-[#161b22]'
                                : 'text-[#8b949e] border-transparent hover:text-[#c9d1d9] hover:bg-[#161b22]/50'"
                        >
                            <span class="text-sm">{{ tab.icon }}</span>
                            <span>{{ tab.label }}</span>
                            <span
                                class="w-2 h-2 rounded-full transition-all"
                                :style="{ backgroundColor: activeTab === tab.id ? tab.color : 'transparent' }"
                            ></span>
                        </button>
                    </div>
                    <div class="flex items-center pr-2 gap-1">
                        <button @click="isEditorMaximized = !isEditorMaximized; isPreviewMaximized = false"
                            class="p-1 text-[#484f58] hover:text-[#8b949e] rounded transition-colors"
                            :title="isEditorMaximized ? 'Restore' : 'Maximize editor'">
                            <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path v-if="!isEditorMaximized" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8V4m0 0h4M4 4l5 5m11-1V4m0 0h-4m4 0l-5 5M4 16v4m0 0h4m-4 0l5-5m11 5l-5-5m5 5v-4m0 4h-4"/>
                                <path v-else stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 9V4.5M9 9H4.5M9 9L3.5 3.5M9 15v4.5M9 15H4.5M9 15l-5.5 5.5M15 9h4.5M15 9V4.5M15 9l5.5-5.5M15 15h4.5M15 15v4.5m0-4.5l5.5 5.5"/>
                            </svg>
                        </button>
                    </div>
                </div>

                <!-- Monaco Editor Container -->
                <div ref="editorContainerRef" class="flex-1 overflow-hidden relative"></div>
            </div>

            <!-- Resizer (Visual) -->
            <div v-if="!isEditorMaximized && !isPreviewMaximized"
                class="shrink-0 group"
                :class="layout === 'horizontal' ? 'w-[3px] cursor-col-resize hover:bg-violet-500/40 bg-[#21262d]' : 'h-[3px] cursor-row-resize hover:bg-violet-500/40 bg-[#21262d]'"
            >
            </div>

            <!-- Preview Panel -->
            <div class="flex flex-col overflow-hidden bg-[#0d1117]"
                :class="[
                    isPreviewMaximized ? 'flex-1' : '',
                    isEditorMaximized ? 'hidden' : '',
                    !isEditorMaximized && !isPreviewMaximized ? (layout === 'horizontal' ? 'w-1/2' : 'h-1/2') : '',
                    isPreviewMaximized ? 'w-full h-full' : ''
                ]"
            >
                <!-- Preview Header -->
                <div class="flex items-center justify-between px-4 py-1.5 bg-[#0d1117] border-b border-[#21262d] shrink-0">
                    <div class="flex items-center gap-2">
                        <div class="flex gap-1.5">
                            <div class="w-2.5 h-2.5 rounded-full bg-[#f85149]"></div>
                            <div class="w-2.5 h-2.5 rounded-full bg-[#d29922]"></div>
                            <div class="w-2.5 h-2.5 rounded-full bg-[#3fb950]"></div>
                        </div>
                        <span class="text-xs text-[#8b949e] font-medium ml-2">Preview</span>
                    </div>
                    <div class="flex items-center gap-1">
                        <button @click="updatePreview" class="p-1 text-[#484f58] hover:text-[#8b949e] rounded transition-colors" title="Refresh preview">
                            <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/></svg>
                        </button>
                        <button @click="isPreviewMaximized = !isPreviewMaximized; isEditorMaximized = false"
                            class="p-1 text-[#484f58] hover:text-[#8b949e] rounded transition-colors"
                            :title="isPreviewMaximized ? 'Restore' : 'Maximize preview'">
                            <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path v-if="!isPreviewMaximized" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 8V4m0 0h4M4 4l5 5m11-1V4m0 0h-4m4 0l-5 5M4 16v4m0 0h4m-4 0l5-5m11 5l-5-5m5 5v-4m0 4h-4"/>
                                <path v-else stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 9V4.5M9 9H4.5M9 9L3.5 3.5M9 15v4.5M9 15H4.5M9 15l-5.5 5.5M15 9h4.5M15 9V4.5M15 9l5.5-5.5M15 15h4.5M15 15v4.5m0-4.5l5.5 5.5"/>
                            </svg>
                        </button>
                    </div>
                </div>

                <!-- Preview iFrame -->
                <div class="flex-1 overflow-hidden bg-white relative">
                    <iframe
                        ref="previewRef"
                        sandbox="allow-scripts allow-modals allow-popups allow-same-origin"
                        class="w-full h-full border-none"
                        title="Live Preview"
                    ></iframe>
                </div>

                <!-- Console Panel -->
                <div v-if="showConsole" class="h-48 flex flex-col bg-[#0d1117] border-t border-[#21262d] shrink-0">
                    <div class="flex items-center justify-between px-3 py-1.5 bg-[#161b22] border-b border-[#21262d]">
                        <div class="flex items-center gap-2">
                            <svg class="w-3.5 h-3.5 text-[#8b949e]" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 9l3 3-3 3m5 0h3"/></svg>
                            <span class="text-xs text-[#8b949e] font-medium">Console</span>
                            <span class="text-[10px] text-[#484f58] bg-[#0d1117] px-1.5 py-0.5 rounded-full font-mono">{{ consoleLogs.length }}</span>
                        </div>
                        <button @click="clearConsole" class="text-[10px] text-[#484f58] hover:text-[#8b949e] transition-colors font-medium uppercase tracking-wider">Clear</button>
                    </div>
                    <div class="flex-1 overflow-y-auto p-2 space-y-0.5 font-mono text-xs">
                        <div v-if="consoleLogs.length === 0" class="text-[#484f58] text-center py-4">No console output yet</div>
                        <div
                            v-for="(log, index) in consoleLogs"
                            :key="index"
                            class="flex items-start gap-2 px-2 py-1 rounded hover:bg-[#161b22] transition-colors"
                            :class="consoleColor(log.method)"
                        >
                            <span class="shrink-0 w-4 text-center">{{ consoleIcon(log.method) }}</span>
                            <span class="flex-1 break-all">{{ log.args.join(' ') }}</span>
                            <span class="text-[10px] text-[#484f58] shrink-0">{{ log.timestamp }}</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Status Bar -->
        <footer class="flex items-center justify-between px-4 h-6 bg-[#0d1117] border-t border-[#21262d] shrink-0 text-[10px] text-[#484f58]">
            <div class="flex items-center gap-3">
                <span class="flex items-center gap-1">
                    <span class="w-1.5 h-1.5 rounded-full bg-emerald-500 animate-pulse"></span>
                    Ready
                </span>
                <span>{{ layout === 'horizontal' ? 'Side by Side' : 'Stacked' }}</span>
            </div>
            <div class="flex items-center gap-3">
                <span>Font: {{ fontSize }}px</span>
                <span>Wrap: {{ wordWrap === 'on' ? 'On' : 'Off' }}</span>
                <span>Monaco Editor</span>
            </div>
        </footer>
    </div>
</template>

<style>
/* Scrollbar global styles */
::-webkit-scrollbar {
    width: 6px;
    height: 6px;
}
::-webkit-scrollbar-track {
    background: transparent;
}
::-webkit-scrollbar-thumb {
    background: #30363d;
    border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
    background: #484f58;
}

/* Monaco overrides */
.monaco-editor .margin {
    background: #0d1117 !important;
}

/* Smooth animations */
* {
    scrollbar-width: thin;
    scrollbar-color: #30363d transparent;
}
</style>
