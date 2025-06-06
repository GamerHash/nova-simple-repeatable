<template>
  <DefaultField
    :field="currentField"
    :errors="errors"
    :show-help-text="showHelpText"
    class="simple-repeatable form-field"
  >
    <template #field>
      <div class="flex flex-col" v-bind="extraAttributes">
        <!-- Title columns -->
        <div v-if="rows.length" class="mb-1 o1-w-full o1-flex o1-border-b o1-py-2 dark:o1-border-slate-600">
          <div
            v-for="(rowField, i) in fields"
            :key="i"
            class="o1-font-bold o1-text-90 o1-text-md o1-w-full o1-ml-3 o1-flex"
            :style="{ maxWidth: rowField.nsrWidth || null }"
          >
            {{ rowField.name }}
            <span v-if="rowField.required" class="o1-text-red-500 o1-text-sm o1-pl-1">
              {{ __('*') }}
            </span>

            <!--  If field is nova-translatable, render separate locale-tabs   -->
            <nova-translatable-locale-tabs
              style="padding: 0"
              class="o1-ml-auto"
              v-if="rowField.component === 'translatable-field'"
              :locales="rowField.formattedLocales"
              :display-type="rowField.translatable.display_type"
              :active-locale="activeLocales[i] || rowField.formattedLocales[0].key"
              :locales-with-errors="repeatableValidation.locales[rowField.originalAttribute]"
              @tabClick="locale => setAllLocales(`sr-${field.attribute}-${rowField.originalAttribute}`, locale)"
              @doubleClick="locale => setAllLocales(void 0, locale)"
            />
          </div>
        </div>

        <draggable
          v-model="rows"
          :item-key="(el, i) => (el && el[0] && el[0].attribute) || i"
          handle=".vue-draggable-handle"
        >
          <template #item="{ element, index }">
            <div class="simple-repeatable-row o1-flex o1-py-2 o1-pl-3 o1-relative o1-rounded-md">
              <div class="vue-draggable-handle o1-flex o1-justify-center o1-items-center o1-cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="22" height="22" class="fill-current">
                  <path
                    d="M4 5h16a1 1 0 0 1 0 2H4a1 1 0 1 1 0-2zm0 6h16a1 1 0 0 1 0 2H4a1 1 0 0 1 0-2zm0 6h16a1 1 0 0 1 0 2H4a1 1 0 0 1 0-2z"
                  />
                </svg>
              </div>

              <div class="simple-repeatable-fields-wrapper o1-w-full o1-flex">
                <component
                  v-for="(rowField, j) in element"
                  :key="j"
                  :is="`form-${rowField.component}`"
                  :field="rowField"
                  :errors="repeatableValidation.errors"
                  :unique-id="getUniqueId(field, rowField)"
                  class="o1-mr-3"
                  :style="{ maxWidth: rowField.nsrWidth || null }"
                />
              </div>

              <div
                class="delete-icon o1-flex o1-justify-center o1-items-center o1-cursor-pointer o1-fill-current hover:o1-fill-red-600"
                @click="deleteRow(index)"
                v-if="canDeleteRows"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="22"
                  height="22"
                  viewBox="0 0 24 24"
                  class="o1-fill-inherit"
                >
                  <path
                    d="M8 6V4c0-1.1.9-2 2-2h4a2 2 0 012 2v2h5a1 1 0 010 2h-1v12a2 2 0 01-2 2H6a2 2 0 01-2-2V8H3a1 1 0 110-2h5zM6 8v12h12V8H6zm8-2V4h-4v2h4zm-4 4a1 1 0 011 1v6a1 1 0 01-2 0v-6a1 1 0 011-1zm4 0a1 1 0 011 1v6a1 1 0 01-2 0v-6a1 1 0 011-1z"
                  />
                </svg>
              </div>
            </div>
          </template>
        </draggable>

        <Button
          v-if="canAddRows"
          @click="addRow"
          :class="{ 'delete-width': canDeleteRows, 'mt-3': rows.length }"
          type="button"
        >
          {{ field.addRowLabel }}
        </Button>
      </div>
    </template>
  </DefaultField>
</template>

<script>
import Draggable from 'vuedraggable';
import { Errors } from 'form-backend-validation';
import { HandlesValidationErrors, DependentFormField } from 'laravel-nova';
import HandlesRepeatable from '../mixins/HandlesRepeatable';
import _set from 'lodash/set';
import { Button } from 'laravel-nova-ui'

export default {
  mixins: [HandlesValidationErrors, HandlesRepeatable, DependentFormField],

  components: {Button, Draggable },

  props: ['resourceName', 'resourceId', 'field'],

  methods: {
    fill(formData) {
      const ARR_REGEX = () => /\[\d+\]$/g;

      const allValues = [];

      for (const row of this.rows) {
        let formData = new FormData();
        const rowValues = {};

        // Fill formData with field values
        row.forEach(field => field.fill(formData));

        // Save field values to rowValues
        for (const item of formData) {
          let normalizedValue = null;

          let key = item[0];
          if (key.split('---').length === 3) {
            key = key.split('---').slice(1).join('---');
          }
          key = key.replace(/---\d+/, '');

          // Is key is an array, we need to remove the '.en' part from '.en[0]'
          const isArray = !!key.match(ARR_REGEX());
          if (isArray) {
            const result = ARR_REGEX().exec(key);
            key = `${key.slice(0, result.index)}${key.slice(result.index + result[0].length)}`;
          }

          try {
            // Attempt to parse value
            normalizedValue = JSON.parse(item[1]);
          } catch (e) {
            // Value is already a valid string
            normalizedValue = item[1];
          }

          if (isArray) {
            if (!rowValues[key]) rowValues[key] = [];
            rowValues[key].push(normalizedValue);
          } else {
            _set(rowValues, key, normalizedValue);
          }
        }

        allValues.push(rowValues);
      }

      formData.append(this.field.attribute, JSON.stringify(allValues));
    },

    addRow() {
      this.rows.push(this.copyFields(this.field.fields, this.rows.length));
    },

    deleteRow(index) {
      this.rows.splice(index, 1);
    },
  },

  computed: {
    extraAttributes() {
      const attrs = this.currentField.extraAttributes;
      return {
        ...attrs,
      };
    },
    repeatableValidation() {
      const fields = this.fields;
      const errors = this.errors.errors;
      const repeaterAttr = this.field.attribute;
      const safeRepeaterAttr = this.field.attribute.replace(/.{16}__/, '');
      const erroredFieldLocales = {};
      const formattedKeyErrors = {};

      // Find errored locales
      for (const field of fields) {
        const fieldAttr = field.originalAttribute;

        // Find all errors related to this field
        const relatedErrors = Object.keys(errors).filter(
          err => !!err.match(new RegExp(`^${safeRepeaterAttr}.\\d+.${fieldAttr}`))
        );

        const isTranslatable = field.component === 'translatable-field';
        if (isTranslatable) {
          const foundLocales = relatedErrors.map(errorKey => errorKey.split('.').slice(-1)).flat();
          erroredFieldLocales[fieldAttr] = foundLocales;
        }

        // Format field
        relatedErrors.forEach(errorKey => {
          const rowIndex = errorKey.split('.')[1];
          let uniqueKey = `${repeaterAttr}---${field.originalAttribute}---${rowIndex}`;

          if (isTranslatable) {
            const locale = errorKey.split('.').slice(-1)[0];
            uniqueKey = `${uniqueKey}.${locale}`;
          }

          formattedKeyErrors[uniqueKey] = errors[errorKey];
        });
      }

      return {
        errors: new Errors(formattedKeyErrors),
        locales: erroredFieldLocales,
      };
    },

    canAddRows() {
      if (!this.currentField.canAddRows) return false;
      if (!!this.currentField.maxRows) return this.rows.length < this.currentField.maxRows;
      return true;
    },

    canDeleteRows() {
      if (!this.currentField.canDeleteRows) return false;
      if (!!this.currentField.minRows) return this.rows.length > this.currentField.minRows;
      return true;
    },
  },
};
</script>

<style lang="scss">
.simple-repeatable.form-field {
  .simple-repeatable-row {
    width: calc(100% + 68px);

    > .simple-repeatable-fields-wrapper {
      .translatable-field {
        padding-top: 0 !important;
      }

      > *,
        // Improve compatibility with nova-translatable
      .translatable-field > div:not(:first-child) > div {
        flex: 1;
        flex-shrink: 0;
        min-width: 0;
        border: none !important;
        padding-top: 0 !important;
        padding-bottom: 0 !important;

        // Hide name
        > *:nth-child(1):not(:only-child) {
          display: none;
        }

        > *:only-child {
          > *:nth-child(1):not(:only-child) {
            display: none;
          }

          > :nth-child(2) {
            width: 100% !important;
            padding: 0 !important;
          }
        }

        // Improve compatibility with nova-compact-theme
        .compact-nova-field-wrapper {
          padding: 0 !important;
        }

        // Fix field width and padding
        > :nth-child(2) {
          width: 100% !important;
          padding: 0 !important;
        }
      }
    }

    margin-left: -46px;

    .delete-icon {
      width: 36px;
      height: 36px;
      margin-right: 10px;
    }

    .vue-draggable-handle {
      height: 36px;
      width: 36px;
      margin-right: 10px;

      &:hover {
        opacity: 0.8;
      }
    }

    &:hover {
      background: var(--40);
    }
  }

  .add-button {
    width: calc(100% + 11px);

    &.delete-width {
      width: calc(100% - 22px);
    }
  }

  > :nth-child(1) {
    min-width: 20%;
  }

  // Make field area full width
  > :nth-child(2) {
    width: 100% !important;
    margin-right: 24px;
  }

  // Compact theme support
  > *:only-child {
    > *:nth-child(1) {
      min-width: 20%;
    }

    // Make field area full width
    > *:nth-child(2) {
      width: 100% !important;
      margin-right: 24px;
    }
  }
}
</style>
