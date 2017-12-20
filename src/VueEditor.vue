<template>
    <div class="quillWrapper">
        <div ref="quillContainer" :id="id"></div>
        <div v-if="counterLimit" class="quillCounter" :id="id+'counter'"></div>
        <input v-if="useCustomImageHandler" @change="emitImageInfo($event)" ref="fileInput" id="file-upload" type="file"
               style="display:none;">
    </div>
</template>

<script>
    import Quill from 'quill'
    import 'quill/dist/quill.core.css'
    import 'quill/dist/quill.snow.css'

    var defaultToolbar = [
        ['bold', 'italic', 'underline', 'strike'],
        ['blockquote', 'code-block', 'image'],

        [{'list': 'ordered'}, {'list': 'bullet'}, {'list': 'check'}],

        [{'indent': '-1'}, {'indent': '+1'}],
        [{'header': [1, 2, 3, 4, 5, 6, false]}],

        [{'color': []}, {'background': []}],
        [{'font': []}],
        [{'align': []}],

        ['clean']
    ]

    class Counter {
        constructor(quill, options) {
            this.quill = quill;
            this.options = options;
            this.container = document.querySelector(options.container);
            quill.on('text-change', this.update.bind(this));
            this.update();  // Account for initial contents
        }

        calculate() {
            let text = this.quill.getText();
            let length = 0;
            if (this.options.unit === 'word') {
                text = text.trim();
                // Splitting empty text returns a non-empty array
                length = text.length > 0 ? text.split(/\s+/).length : 0;
            } else {
                length = text.length - 1;
            }
            if (length > this.options.limit) {
                if (this.options.unit === 'word') {
                    this.quill.setText(text.split(/\s+/).slice(0, this.options.limit).join(" "))
                    text = this.quill.getText().trim();
                    length = text.length > 0 ? text.split(/\s+/).length : 0;
                }
                else {
                    this.quill.deleteText(this.options.limit, length);
                    length = this.quill.getLength() - 1;
                }
            }
            return length;
        }

        update() {
            var length = this.calculate();
            var label = this.options.unit;
            // if (length !== 1) {
            label += 's';
            // }
            this.container.innerText = length + ' / ' + this.options.limit + ' ' + label;
        }
    }

    Quill.register('modules/counter', Counter);

    export default {
        name: 'vue-editor',
        props: {
            counterLimit: {
                type: Number,
                default: 0,
            },
            counterType: {
                type: String,
                default: 'word'
            },
            value: String,
            id: {
                type: String,
                default: 'quill-container'
            },
            placeholder: String,
            disabled: Boolean,
            editorToolbar: Array,
            useCustomImageHandler: {
                type: Boolean,
                default: false
            }
        },

        data() {
            return {
                quill: null,
                editor: null,
                toolbar: this.editorToolbar ? this.editorToolbar : defaultToolbar,
            }
        },

        mounted() {
            this.initializeVue2Editor()
            this.handleUpdatedEditor()
        },

        watch: {
            value(val) {
                if (val != this.editor.innerHTML && !this.quill.hasFocus()) {
                    this.editor.innerHTML = val
                }
            },
            disabled(status) {
                this.quill.enable(!status);
            }
        },

        methods: {
            initializeVue2Editor() {
                this.setQuillElement()
                this.setEditorElement()
                this.checkForInitialContent()
            },

            setQuillElement() {
                var modules = {
                    toolbar: this.toolbar
                };
                if (this.counterLimit) {
                    Quill.register('modules/counter', Counter);
                    modules.counter = {
                        container: '#' + this.id + 'counter',
                        unit: this.counterType,
                        limit: this.counterLimit,
                    }
                }
                this.quill = new Quill(this.$refs.quillContainer, {
                    modules: modules,
                    placeholder: this.placeholder ? this.placeholder : '',
                    theme: 'snow',
                    readOnly: this.disabled ? this.disabled : false,
                })
                this.checkForCustomImageHandler()
            },

            setEditorElement() {
                this.editor = document.querySelector(`#${this.id} .ql-editor`)
            },

            checkForInitialContent() {
                this.editor.innerHTML = this.value || ''
            },

            checkForCustomImageHandler() {
                this.useCustomImageHandler === true ? this.setupCustomImageHandler() : ''
            },

            setupCustomImageHandler() {
                let toolbar = this.quill.getModule('toolbar');
                toolbar.addHandler('image', this.customImageHandler);
            },

            handleUpdatedEditor() {
                this.quill.on('text-change', () => {
                    this.$emit('input', this.editor.innerHTML)
                })
            },

            customImageHandler(image, callback) {
                this.$refs.fileInput.click();
            },

            emitImageInfo($event) {
                let file = $event.target.files[0]
                let Editor = this.quill
                let range = Editor.getSelection();
                let cursorLocation = range.index
                this.$emit('imageAdded', file, Editor, cursorLocation)
            }
        }
    }
</script>

<style>
    .ql-editor {
        min-height: 200px;
        /*font-size: 16px;*/
    }

    .ql-snow .ql-thin, .ql-snow .ql-stroke.ql-thin {
        stroke-width: 1px !important;
    }

    .quillWrapper .ql-snow.ql-toolbar {
        padding-top: 8px;
        padding-bottom: 4px;
    }

    .quillWrapper .ql-snow.ql-toolbar button {
        margin: 1px;
    }

    .quillWrapper .ql-snow.ql-toolbar .ql-formats {
        margin-bottom: 10px;
    }

    .quillWrapper .ql-snow.ql-toolbar button svg, .ql-snow .ql-toolbar button svg {
        width: 22px;
        height: 22px;
    }

    .quillWrapper .ql-editor ul[data-checked=true] > li::before, .quillWrapper .ql-editor ul[data-checked=false] > li::before {
        font-size: 1.35em;
        vertical-align: baseline;
        bottom: -0.065em;
        font-weight: 900;
        color: #222;
    }

    .quillWrapper .ql-snow .ql-stroke {
        stroke: rgba(63, 63, 63, 0.95);
        stroke-linecap: square;
        stroke-linejoin: initial;
        stroke-width: 1.7px;
    }

    .quillWrapper .ql-picker-label {
        font-size: 15px;
    }

    .quillWrapper .ql-snow .ql-active .ql-stroke {
        stroke-width: 2.25px;
    }

    .quillWrapper .ql-toolbar.ql-snow .ql-formats {
        vertical-align: top;
    }

    .quillWrapper .quillCounter {
        border: 1px solid #ccc;
        border-width: 0px 1px 1px 1px;
        color: #aaa;
        padding: 5px 15px;
        text-align: right;
    }

</style>
