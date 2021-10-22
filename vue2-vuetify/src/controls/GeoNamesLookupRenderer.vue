<template>
  <control-wrapper
    v-bind="controlWrapper"
    :styles="styles"
    :isFocused="isFocused"
    :appliedOptions="appliedOptions"
  >
    <h6>Geonames Lookup</h6>
    <v-text-field
      type="number"
      :step="step"
      :id="control.id + '-input'"
      :class="styles.control.input"
      :disabled="!control.enabled"
      :autofocus="appliedOptions.focus"
      :placeholder="appliedOptions.placeholder"
      :label="computedLabel"
      :hint="control.description"
      :persistent-hint="persistentHint()"
      :required="control.required"
      :error-messages="control.errors"
      :value="control.data"
      @change="onChange"
      @focus="isFocused = true"
      @blur="isFocused = false"
    />
  </control-wrapper>
</template>

<script lang="ts">
import {
  ControlElement,
  Dispatch,
  Actions,
  JsonFormsRendererRegistryEntry,
  rankWith,
  isIntegerControl,
  and,
  Tester,
  optionIs,
} from '@jsonforms/core';
import { defineComponent, inject } from '../vue';
import {
  rendererProps,
  useJsonFormsControl,
  RendererProps,
} from '@jsonforms/vue2';
import { default as ControlWrapper } from './ControlWrapper.vue';
import { useVuetifyControl } from '../util';
import { VTextField } from 'vuetify/lib';
import { CoreActions } from '@jsonforms/core';

const controlRenderer = defineComponent({
  name: 'geonames-lookup-renderer',
  components: {
    ControlWrapper,
    VTextField,
  },
  props: {
    ...rendererProps<ControlElement>(),
  },
  setup(props: RendererProps<ControlElement>) {
    const dispatch = inject<Dispatch<CoreActions>>('dispatch');
    const vControl = useVuetifyControl(
      useJsonFormsControl(props),
      (value) => parseInt(value, 10) || undefined
    );
    return { ...vControl, dispatch };
  },
  methods: {
    mapDict() {
      return [
        { path: 'addresses.lat', geoname: 'lat', type: 'float' },
        { path: 'addresses.lng', geoname: 'lng', type: 'float' },
        {
          path: 'addresses.country_geonames_id',
          geoname: 'countryId',
          type: 'int',
        },
        {
          path: 'addresses.geonames_city.city',
          geoname: 'name',
          type: 'string',
        },
      ];
    },
    checkData(data: string, type: string) {
      switch (type) {
        case 'float':
          return parseFloat(data);
        case 'int':
          return parseInt(data);
        default:
          return data;
      }
    },
    fetchAddress(id: string) {
      const url = new URL('http://api.geonames.org/getJSON');
      const params = { geonameId: id, username: 'roradmin' }; // or:
      url.search = new URLSearchParams(params).toString();
      fetch(url.toString()).then((response) => {
        response.json().then((data) => {
          if (this.dispatch) {
            let mapping = this.mapDict();
            for (const entry of mapping) {
              this.dispatch(
                Actions.update(entry.path, () =>
                  this.checkData(data[entry.geoname], entry.type)
                )
              );
            }
          }
        });
      });
    },
    onChange(e: number) {
      console.log(e);
      const id = e.toString();
      this.fetchAddress(id);
    },
  },
  computed: {
    step(): number {
      const options: any = this.appliedOptions;
      return options.step ?? 1;
    },
  },
});

export default controlRenderer;

const locationIDTester: Tester = and(
  isIntegerControl,
  optionIs('locationID', true)
);

export const entry: JsonFormsRendererRegistryEntry = {
  renderer: controlRenderer,
  tester: rankWith(10, locationIDTester),
};
</script>
