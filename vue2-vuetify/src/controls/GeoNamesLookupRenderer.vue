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
  JsonFormsSubStates,
  Dispatch,
  Actions,
  JsonSchema,
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
import set from 'lodash/fp/set';

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
    const jsonforms = inject<JsonFormsSubStates>('jsonforms');
    const s = jsonforms?.core?.schema;
    const ui = jsonforms?.core?.uischema;
    const vControl = useVuetifyControl(
      useJsonFormsControl(props),
      (value) => parseInt(value, 10) || undefined
    );
    return { ...vControl, dispatch, jsonforms, s, ui };
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
        {
          path: 'addresses.city',
          geoname: 'name',
          type: 'string',
        },
        {
          path: 'country.country_name',
          geoname: 'countryName',
          type: 'string',
        },
        {
          path: 'country.country_code',
          geoname: 'countryCode',
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
    processGeoNamesAdmin(level: string) {
      return [
        { ror: 'name', geoname: 'adminName' + level, type: 'string' },
        { ror: 'ascii_name', geoname: 'adminName' + level, type: 'string' },
        { ror: 'id', geoname: 'adminId' + level, type: 'int' },
      ];
    },
    fetchAddress(id: string) {
      const url = new URL('http://api.geonames.org/getJSON');
      const params = { geonameId: id, username: 'roradmin' }; // or:
      url.search = new URLSearchParams(params).toString();
      const rootData = this.jsonforms?.core?.data;
      fetch(url.toString()).then((response) => {
        response.json().then((data) => {
          if (this.dispatch) {
            let mapping = this.mapDict();
            let updatedData = rootData;
            updatedData = set(
              'addresses.geonames_city.id',
              parseInt(id),
              updatedData
            );
            for (const entry of mapping) {
              updatedData = set(
                entry.path,
                this.checkData(data[entry.geoname], entry.type),
                updatedData
              );
            }
            if (data.adminId1) {
              let path = 'addresses.geonames_city.geonames_admin1.';
              let geonamesAdmin1 = this.processGeoNamesAdmin('1');
              for (const admin of geonamesAdmin1) {
                updatedData = set(
                  path + admin.ror,
                  this.checkData(data[admin.geoname], admin.type),
                  updatedData
                );
              }
              updatedData = set(
                path + 'code',
                [data.countryCode, data.adminCode1].join('.'),
                updatedData
              );
              if (data.adminId2) {
                let path = 'addresses.geonames_city.geonames_admin2.';
                let geonamesAdmin2 = this.processGeoNamesAdmin('2');
                for (const admin of geonamesAdmin2) {
                  updatedData = set(
                    path + admin.ror,
                    this.checkData(data[admin.geoname], admin.type),
                    updatedData
                  );
                }
                updatedData = set(
                  path + 'code',
                  [data.countryCode, data.adminCode1, data.adminCode2].join(
                    '.'
                  ),
                  updatedData
                );
              }
            }
            this.dispatch(
              Actions.updateCore(updatedData, this.s as JsonSchema, this.ui)
            );
          }
        });
      });
    },
    onChange(e: number) {
      const id = e.toString();
      this.fetchAddress(id);
      this.control.data = e;
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
