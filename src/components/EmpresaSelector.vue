<template>
  <v-select
    v-model="empresaActiva"
    :items="empresas"
    item-title="nombreFantasia"
    item-value="_id"
    label="Empresa"
    prepend-inner-icon="mdi-office-building"
    outlined
    dense
    hide-details
    clearable
    @update:model-value="cambiarEmpresa"
    @click:clear="seleccionarTodas"
  >
    <template #prepend-item>
      <v-list-item @click="seleccionarTodas" base-color="primary">
        <v-list-item-title class="d-flex align-center">
          <v-icon start size="small">mdi-check</v-icon>
          Todas las empresas
        </v-list-item-title>
      </v-list-item>
      <v-divider class="my-1"></v-divider>
    </template>

    <template #selection="{ item }">
      <div class="d-flex align-center">
        <v-icon start size="small">mdi-office-building</v-icon>
        <span class="text-truncate">{{ item.title }}</span>
      </div>
    </template>
    <template #item="{ props, item }">
      <v-list-item v-bind="props">
        <template #prepend>
          <v-icon>mdi-office-building</v-icon>
        </template>
        <template #title>{{ item.title }}</template>
        <template #subtitle>
          <div class="d-flex align-center">
            <v-icon start size="x-small">mdi-numeric</v-icon>
            RUC: {{ item.raw.ruc }}
          </div>
        </template>
      </v-list-item>
    </template>
  </v-select>
</template>

<script>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import { guardarEmpresaActiva, obtenerEmpresaActiva, limpiarEmpresaActiva } from '../auth';

export default {
  name: 'EmpresaSelector',
  emits: ['cambio-empresa'],
  setup(props, { emit }) {
    const empresas = ref([]);
    const empresaActiva = ref(null);
    const cargando = ref(false);

    // Computed que agrega "Todas las empresas" como primera opción
    const empresasConTodas = computed(() => [
      { _id: null, nombreFantasia: 'Todas las empresas' }
    ].concat(empresas.value));

    const cargarEmpresas = async () => {
      cargando.value = true;
      try {
        const response = await axios.get('/api/empresas');
        empresas.value = response.data.data || [];
        
        // Restaurar empresa activa guardada (formato RUC)
        const guardada = obtenerEmpresaActiva();
        console.log('Empresa guardada en localStorage:', guardada);
        if (guardada === 'all' || guardada === null) {
          empresaActiva.value = null; // Todas las empresas
        } else if (guardada && empresas.value.find(e => e.ruc === guardada)) {
          const empresa = empresas.value.find(e => e.ruc === guardada);
          empresaActiva.value = empresa._id;
        }
      } catch (error) {
        console.error('Error cargando empresas:', error);
      } finally {
        cargando.value = false;
      }
    };

    const seleccionarTodas = () => {
      empresaActiva.value = null;
      guardarEmpresaActiva('all');
      emit('cambio-empresa', 'all');
    };

    const cambiarEmpresa = (id) => {
      if (id === null || id === undefined) {
        seleccionarTodas();
        return;
      }
      const empresa = empresas.value.find(e => e._id === id);
      if (empresa) {
        guardarEmpresaActiva(empresa.ruc);
        emit('cambio-empresa', empresa.ruc);
      }
    };

    onMounted(() => {
      cargarEmpresas();
    });

    return {
      empresas: empresasConTodas,
      empresaActiva,
      cargando,
      cambiarEmpresa,
      seleccionarTodas
    };
  }
};
</script>

<style scoped>
/* Ajustes para que el selector se vea bien en el app bar */
:deep(.v-field__input) {
  min-height: 40px !important;
}

:deep(.v-selection-control) {
  flex: none !important;
}
</style>