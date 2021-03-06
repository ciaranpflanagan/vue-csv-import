<template>
  <div class="vue-csv-uploader">
    <div class="form">
      <div class="vue-csv-uploader-part-one">
        <div class="form-check form-group csv-import-checkbox" v-if="headers === null" style="margin-bottom: 1em;">
          <slot name="hasHeaders" :headers="hasHeaders" :toggle="toggleHasHeaders">
            <input :class="checkboxClass" type="checkbox" :id="makeId('hasHeaders')" :value="hasHeaders" @change="toggleHasHeaders">
            <label class="form-check-label" :for="makeId('hasHeaders')">
                Select this box if your file has column names.
            </label>
          </slot>
        </div>
        <div class="form-group csv-import-file" style="margin-bottom: 1em;">
          <div class="file has-name">
            <label class="file-label" style="justify-content: center;">
              <input ref="csv" type="file" @change.prevent="validFileMimeType" class="file-input" name="csv">
              <span class="file-cta">
                <span class="file-icon">
                  <i class="fas fa-upload"></i>
                </span>
                <span class="file-label">
                  {{ this.filename }}
                </span>
              </span>
              <!-- Todo - get filename -->
             <!--  <span class="file-name" :key="filename">
                {{ this.filename }}
              </span> -->
            </label>
          </div>

          <slot name="error" v-if="showErrorMessage">
            <div class="invalid-feedback d-block">
                File type is invalid
            </div>
          </slot>
        </div>
        <div class="form-group">
          <slot name="next" :load="load">
            <button 
                :disabled="disabledNextButton"
                class="button is-medium is-primary"
                @click.prevent="load"
                v-show="nextClicked"
            >{{ loadBtnText }}</button>
          </slot>
        </div>
      </div>
      <div class="vue-csv-uploader-part-two">
        <div class="vue-csv-mapping" v-if="sample">
            <div class="columns">
                <div class="column is-half">
                    <p>Field</p>
                </div>
                <div class="column is-half">
                    <p>CSV Column</p>
                </div>
            </div>
            <!-- There was a table here but we needed to split it into 2 columns -->
            <div class="columns">
                <div class="column is-half">
                    <div v-for="(field, key) in fieldsToMap.slice(0,10)" :key="key" class="flex">
                        <div class="flex-05">{{ field.label }}</div>
                        <div class="flex-1 mb1em">
                          <select class="input" :name="`csv_uploader_map_${key}`" v-model="map[field.key]">
                            <option :value="null" disabled hidden>Select a column</option>
                            <option v-for="(column, key) in firstRow" :key="key" :value="key">{{ column }}</option>
                          </select>
                        </div>
                    </div>
                </div>
                <div class="column is-half">
                    <div v-for="(field, key) in fieldsToMap.slice(11,20)" :key="key" class="flex">
                        <div class="flex-05">{{ field.label }}</div>
                        <div class="flex-1 mb1em">
                          <select class="input" :name="`csv_uploader_map_${key}`" v-model="map[field.key]">
                            <option :value="null" disabled hidden>Select a column</option>
                            <option v-for="(column, key) in firstRow" :key="key" :value="key">{{ column }}</option>
                          </select>
                        </div>
                    </div>
                </div>
            </div>
          <input type="checkbox" name="termslabel" id="termslabel" v-model="terms" style="margin-right: 1em;" />
          <label for="termslabel">{{ TosText }}</label>

          <div class="form-group mt1em" v-if="url">
            <slot name="submit" :submit="submit">
              <button 
                class="button is-medium is-primary" 
                @click.prevent="submit" 
                :disabled="! terms"
              >Next</button>
            </slot>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { drop, every, forEach, get, isArray, map, set } from 'lodash';
import axios from 'axios';
import Papa from 'papaparse';
import mimeTypes from "mime-types";
export default {
    props: {
        value: Array,
        url: {
            type: String
        },
        mapFields: {
            required: true
        },
        callback: {
            type: Function,
            default: () => ({})
        },
        catch: {
            type: Function,
            default: () => ({})
        },
        finally: {
            type: Function,
            default: () => ({})
        },
        parseConfig: {
            type: Object,
            default() {
                return {};
            }
        },
        headers: {
            default: null
        },
        loadBtnText: {
            type: String,
            default: "Next"
        },
        submitBtnText: {
            type: String,
            default: "Submit"
        },
        tableClass: {
            type: String,
            default: "table"
        },
        checkboxClass: {
            type: String,
            default: "form-check-input"
        },
        buttonClass: {
            type: String,
            default: "btn btn-primary"
        },
        inputClass: {
            type: String,
            default: "form-control-file"
        },
        validation: {
            type: Boolean,
            default: true,
        },
        fileMimeTypes: {
            type: Array,
            default: () => {
                return ["text/csv", "text/x-csv", "application/vnd.ms-excel", "text/plain"];
            }
        },
        TosText: {
            type: String,
            default: "You accept the ToS and acknowledge that your account will be flagged and blocked if you import contacts without their permissions."
        }
    },
    data: () => ({
        form: {
            csv: null,
        },
        fieldsToMap: [],
        map: {},
        hasHeaders: true,
        csv: null,
        sample: null,
        isValidFileMimeType: false,
        fileSelected: false,
        filename: 'Choose a file',
        terms: false,
        nextClicked: true
    }),
    created() {
        this.hasHeaders = this.headers;
        if (isArray(this.mapFields)) {
            this.fieldsToMap = map(this.mapFields, (item) => {
                return {
                    key: item,
                    label: item
                };
            });
        } else {
            this.fieldsToMap = map(this.mapFields, (label, key) => {
                return {
                    key: key,
                    label: label
                };
            });
        }
    },
    methods: {
        submit() {
            console.log('vue-csv.submit');
            const _this = this;
            this.form.csv = this.buildMappedCsv();
            this.$emit('input', this.form.csv);
            // go to add Seqs or Tags
            // if (this.url) {
            //     axios.post(this.url, this.form).then(response => {
            //         _this.callback(response);
            //     }).catch(response => {
            //         _this.catch(response);
            //     }).finally(response => {
            //         _this.finally(response);
            //     });
            // } else {
            //     _this.callback(this.form.csv);
            // }
        },
        buildMappedCsv() {
            const _this = this;
            let csv = this.hasHeaders ? drop(this.csv) : this.csv;
            return map(csv, (row) => {
                let newRow = {};
                forEach(_this.map, (column, field) => {
                    set(newRow, field, get(row, column));
                });
                return newRow;
            });
        },
        validFileMimeType() {
            let file = this.$refs.csv.files[0];
            // this.$emit('testme', file);
            this.filename = file.name;

            const mimeType = file.type === "" ? mimeTypes.lookup(file.name) : file.type;
            if (file) {
                this.fileSelected = true;
                this.isValidFileMimeType = this.validation ? this.validateMimeType(mimeType) : true;
            } else {
                this.isValidFileMimeType = !this.validation;
                this.fileSelected = false;
            }
        },
        validateMimeType(type) {
            return this.fileMimeTypes.indexOf(type) > -1;
        },
        load() {
            const _this = this;

            this.$store.commit('changeModalMargin', true);
            this.nextClicked = false;

            this.readFile((output) => {
                _this.sample = get(Papa.parse(output, { preview: 2, skipEmptyLines: true }), "data");
                _this.csv = get(Papa.parse(output, { skipEmptyLines: true }), "data");
            });
        },
        readFile(callback) {
            let file = this.$refs.csv.files[0];
            if (file) {
                let reader = new FileReader();
                reader.readAsText(file, "UTF-8");
                reader.onload = function (evt) {
                    callback(evt.target.result);
                };
                reader.onerror = function () {
                };
            }
        },
        toggleHasHeaders() {
            this.hasHeaders = !this.hasHeaders;
        },
        makeId(id) {
            return `${id}${this._uid}`;
        }
    },
    watch: {
        map: {
            deep: true,
            handler: function (newVal) {
                if (!this.url) {
                    let hasAllKeys = Array.isArray(this.mapFields) ? every(this.mapFields, function (item) {
                        return newVal.hasOwnProperty(item);
                    }) : every(this.mapFields, function (item, key) {
                        return newVal.hasOwnProperty(key);
                    });
                    if (hasAllKeys) {
                        this.submit();
                    }
                }
            }
        }
    },
    computed: {
        firstRow() {
            return get(this, "sample.0");
        },
        showErrorMessage() {
            return this.fileSelected && !this.isValidFileMimeType;
        },
        disabledNextButton() {
            return !this.isValidFileMimeType;
        }
    },
};
</script>