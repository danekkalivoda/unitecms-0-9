<template>
    <div class="uk-margin">
        <div class="uk-form-controls">
            <div class="uk-form-custom">
                <input v-if="widgetType === 'Symfony\\Component\\Form\\Extension\\Core\\Type\\TextType'" :readonly="auto" :class="{'read-only': auto}" :id="input_id + '_text'" type="text" class="uk-input" :name="name + '[text]'" v-model="text" />
                <textarea v-else :readonly="auto" :class="{'read-only': auto}" :id="input_id + '_text'" type="text" class="uk-textarea" :name="name + '[text]'" v-model="text"></textarea>
            </div>

            <label :for="input_id + '_auto'" class="uk-button uk-button-small" :class="{ 'uk-button-secondary': auto, 'uk-button-default': !auto }">
                <input :id="input_id + '_auto'" type="checkbox" class="uk-checkbox" :name="name + '[auto]'" v-model="auto" />
                {{ auto ? fieldLabels.auto : fieldLabels.manual }}
                <span v-if="auto" v-html="feather.icons['check'].toSvg({ width: 16, height: 16 })"></span>
            </label>

            <p class="uk-form-help-inline">
                <span v-if="willUpdateText && auto">{{ fieldLabels.desc_auto_update }}</span>
                <span v-else-if="!willUpdateText && auto">{{ fieldLabels.desc_no_auto_update }}</span>
                <span v-else>{{ fieldLabels.desc_manual }}</span>
            </p>
        </div>
    </div>
</template>

<script>

    import feather from 'feather-icons';

    export default {
        data() {
            return {
                loading: false,
                form: null,
                input_id: this.name.replace('[', '_').replace(']', '_') + 'url',
                text: this.textValue,
                auto: !!this.autoValue,
                feather: feather,
                willUpdateText: this.updateText || (!this.autoValue),
                fieldLabels: JSON.parse(this.labels),
                queryModifier: [],
            };
        },

        mounted() {
            this.queryModifier = this.collectQueryModifier(this.$el);
            this.form = document.querySelector('form[name="fieldable_form"]');

            // Listen to all actual form elements.
            this.form.querySelectorAll('input, select, textarea').forEach((element) =>{
                element.addEventListener('change', this.generateAutoText);
            });

            // Listen to all dynamic form elements.
            let observer = new MutationObserver((mutationsList) => {
                for(let mutation of mutationsList) {
                    mutation.addedNodes.forEach((node) => {
                        if(node instanceof HTMLElement) {
                            node.querySelectorAll('input, select, textarea').forEach((element) => {
                                element.addEventListener('change', this.generateAutoText);
                            });
                        }
                    });
                    mutation.removedNodes.forEach((node) => {
                        if(node instanceof HTMLElement) {
                            node.querySelectorAll('input, select, textarea').forEach((element) => {
                                element.removeEventListener('change', this.generateAutoText);
                            });
                        }
                    });
                }
            });
            observer.observe(this.form, { childList: true, subtree: true });
        },

        props: [
            'labels',
            'name',
            'textValue',
            'autoValue',
            'widgetType',
            'generationUrl',
            'updateText',
            'contentId',
        ],

        watch: {
            auto(auto) {
                if(auto) {
                    if(this.willUpdateText) {
                        this.generateAutoText();
                    } else {
                        this.text = this.textValue;
                    }
                }
            }
        },
        methods: {
            collectQueryModifier($el) {
                if($el.tagName === 'FORM') {
                    return [];
                }
                let modifiers = [];
                if($el.dataset.graphqlQueryMapper) {
                    let parts = $el.dataset.graphqlQueryMapper.split('=');
                    modifiers.push(parts);
                }
                return modifiers.concat(this.collectQueryModifier($el.parentElement));
            },
            wrapNestedFieldName(fieldName) {
                let parts = fieldName.split('][');
                let rootPart = parts.shift();
                if(parts.length === 0) {
                    return rootPart;
                }

                parts = parts.filter((part) => {
                    return isNaN(part);
                }).map((part) => {
                    let foundPart = this.queryModifier.filter((m) => { return m[0] === part });
                    if(foundPart.length > 0) {
                        return foundPart.pop()[1];
                    } else {
                        return part;
                    }
                });

                return this.wrapNestedFieldName(rootPart + '{'+(parts.join(']['))+'}');
            },

            queryFromFieldName(fieldName) {
                fieldName = fieldName.replace('fieldable_form[', 'query{');
                fieldName = fieldName.substr(0, fieldName.length - 1);
                fieldName = fieldName + '{text_generated}}';
                return this.wrapNestedFieldName(fieldName);
            },
            findNestedValue(result, parentName = null) {
                if(typeof result === 'object') {
                    let objectKeys = Object.keys(result);
                    if(objectKeys.length === 1) {
                        return this.findNestedValue(result[objectKeys[0]], objectKeys[0]);
                    } else {

                        // TODO: This is a little hack for arrays of content (e.g. collection fields). In the future we should improve this!
                        if(this.name.split(parentName + '][').length > 1) {
                            let id = this.name.split(parentName + '][')[1].split('][')[0];
                            if(result[id]) {
                                return this.findNestedValue(result[id], id);
                            }
                        }
                        return this.findNestedValue(result[objectKeys[0]], objectKeys[0]);
                    }
                }
                return result;
            },

            generateAutoText() {
                if(this.auto && this.willUpdateText && !this.loading) {

                    this.loading = true;
                    let queryUrl = this.generationUrl + '?query=' + this.queryFromFieldName(this.name);

                    if(this.contentId) {
                        queryUrl += '&id=' + this.contentId;
                    }

                    let request = new XMLHttpRequest();
                    request.onload = () => {
                        this.text = this.findNestedValue(JSON.parse(request.responseText));
                        this.loading = false;
                    };

                    request.open("POST", queryUrl, true);

                    let formData = new FormData(this.form);
                    formData.append('fieldable_form[submit]', '');
                    request.send(formData);
                }
            }
        }

    }
</script>

<style lang="scss" scoped>
    .uk-form-controls {
        label.uk-button {
            position: relative;
            padding: 0 5px 0 10px;
            flex-shrink: 0;
            margin-top: 5px;
            font-size: 12px;
            text-transform: uppercase;
            line-height: 24px;

            &.uk-button-default {
                background: white;
                padding: 0 10px;
                box-shadow: none;
                border-color: #e5e5e5;
                color: #999;
            }

            input {
                position: absolute;
                top: 0;
                left: 0;
                opacity: 0;
            }
        }

        .uk-form-custom {
            width: 100%;

            textarea {
                resize: vertical;
            }
        }

        @media (min-width: 600px) {
            .uk-form-controls {
                position: relative;

                input, textarea {
                    padding-right: 100px;
                }

                label.uk-button {
                    position: absolute;
                    z-index: 20;
                    right: 7px;
                    top: 2px;
                }
            }
        }
    }
</style>
