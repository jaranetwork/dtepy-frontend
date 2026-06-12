<template>
  <v-container fluid>
    <v-card>
      <v-card-title>
        <h2>Registros de Operación</h2>
        <v-spacer></v-spacer>
        <v-select
          v-model="selectedFilter"
          :items="filterOptions"
          label="Filtrar por estado"
          style="max-width: 200px;"
          density="compact"
        ></v-select>
        <v-text-field
          v-model="search"
          append-inner-icon="mdi-magnify"
          label="Buscar"
          single-line
          hide-details
          style="max-width: 300px;"
          class="ml-4"
        ></v-text-field>
      </v-card-title>

      <v-card-text>
        <v-data-table
          :headers="headers"
          :items="logs"
          :search="search"
          :loading="loading"
          :items-per-page="itemsPerPage"
          hide-default-footer
          class="elevation-1"
        >
          <template v-slot:item.invoiceId.correlativo="{ item }">
            <span v-if="item.invoiceId?.correlativo">{{ item.invoiceId.correlativo }}</span>
            <span v-else class="text-grey">N/A</span>
          </template>

          <template v-slot:item.invoiceId.cdc="{ item }">
            <span v-if="item.invoiceId?.cdc" style="font-family: monospace; font-size: 0.85em;">
              {{ item.invoiceId.cdc }}
            </span>
            <span v-else class="text-grey">N/A</span>
          </template>

          <template v-slot:item.tipoOperacion="{ item }">
            <v-chip
              :color="getLogStatusColor(item.tipoOperacion, item.estado)"
              size="small"
              variant="flat"
            >
              {{ item.tipoOperacion }}
            </v-chip>
          </template>

          <template v-slot:item.estado="{ item }">
            <v-chip
              :color="getLogStateColor(item.estado)"
              size="small"
              variant="flat"
            >
              {{ item.estado }}
            </v-chip>
          </template>

          <template v-slot:item.fecha="{ item }">
            {{ formatDateTime(item.fecha) }}
          </template>

          <template v-slot:item.invoiceId="{ item }">
            <v-btn
              v-if="item.invoiceId"
              color="primary"
              size="small"
              variant="text"
              @click="viewInvoice(item.invoiceId._id)"
            >
              Ver Factura
            </v-btn>
            <span v-else class="text-grey">N/A</span>
          </template>
        </v-data-table>
        
        <div class="d-flex align-center justify-center pt-4" style="gap: 16px;">
          <div class="d-flex align-center" style="gap: 8px;">
            <span class="text-body-2">Items por página:</span>
            <v-select
              v-model="itemsPerPage"
              :items="[10, 15, 25, 50, 100]"
              density="compact"
              variant="outlined"
              hide-details
              style="max-width: 80px;"
            ></v-select>
          </div>
          <v-pagination
            v-model="currentPage"
            :length="totalPages"
            @update:modelValue="changePage"
          ></v-pagination>
        </div>
      </v-card-text>
    </v-card>

  </v-container>
</template>

<script>
import { ref, onMounted, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import axios from 'axios';

export default {
  name: 'LogsView',
  setup() {
    const route = useRoute();
    const router = useRouter();

    const logs = ref([]);
    const loading = ref(true);
    const search = ref('');
    const currentPage = ref(parseInt(route.query.page) || 1);
    const totalPages = ref(1);
    const itemsPerPage = ref(parseInt(route.query.limit) || 15);
    const selectedFilter = ref(route.query.estado || 'all');

    const filterOptions = [
      { title: 'Todos', value: 'all' },
      { title: 'Éxito', value: 'success' },
      { title: 'Error', value: 'error' },
      { title: 'Advertencia', value: 'warning' }
    ];
    
    const headers = [
      { title: 'Correlativo', key: 'invoiceId.correlativo' },
      { title: 'CDC', key: 'invoiceId.cdc' },
      { title: 'Tipo', key: 'tipoOperacion' },
      { title: 'Descripción', key: 'descripcion' },
      { title: 'Estado', key: 'estado' },
      { title: 'Fecha', key: 'fecha' },
      { title: 'Acciones', key: 'invoiceId', sortable: false }
    ];
    
    const getLogStatusColor = (tipo, estado) => {
      // Si el estado es error, mostrar en rojo independientemente del tipo
      if (estado === 'error') {
        return 'error';
      }
      if (estado === 'warning') {
        return 'warning';
      }

      switch(tipo) {
        case 'envio_exitoso':
        case 'inicio_proceso':
        case 'consulta_estado':
          return 'success';
        case 'error':
        case 'error_consulta_estado':
        case 'error_respuesta_set':
          return 'error';
        case 'envio_sifen':
        case 'respuesta_sifen':
          return 'primary';
        default:
          return 'info';
      }
    };
    
    const getLogStateColor = (estado) => {
      switch(estado) {
        case 'success':
          return 'success';
        case 'error':
          return 'error';
        case 'warning':
          return 'warning';
        default:
          return 'info';
      }
    };
    
    const formatDateTime = (dateString) => {
      const date = new Date(dateString);
      return date.toLocaleString('es-PY');
    };
    
    const viewInvoice = (id) => {
      window.location.href = `/invoices/${id}`;
    };

    const updateUrl = () => {
      const query = {};
      if (currentPage.value > 1) query.page = currentPage.value;
      if (itemsPerPage.value !== 15) query.limit = itemsPerPage.value;
      if (selectedFilter.value !== 'all') query.estado = selectedFilter.value;
      router.replace({ query }).catch(() => {});
    };

    const changePage = (page) => {
      currentPage.value = page;
    };
    
    const loadLogs = async () => {
      loading.value = true;
      try {
        const params = new URLSearchParams();
        params.append('page', currentPage.value);
        params.append('limit', itemsPerPage.value);
        if (selectedFilter.value !== 'all') params.append('estado', selectedFilter.value);
        const response = await axios.get(`/api/logs?${params.toString()}`);
        logs.value = response.data.logs || [];
        totalPages.value = response.data.totalPages || 1;
      } catch (error) {
        console.error('Error cargando logs:', error);
        // Datos de ejemplo para mostrar la funcionalidad
        logs.value = [
          {
            _id: '1',
            invoiceId: 'inv123',
            tipoOperacion: 'inicio_proceso',
            descripcion: 'Inicio del proceso de generación de factura electrónica',
            estado: 'success',
            fecha: new Date().toISOString()
          },
          {
            _id: '2',
            invoiceId: 'inv123',
            tipoOperacion: 'envio_sifen',
            descripcion: 'Envío de factura a SIFEN',
            estado: 'success',
            fecha: new Date().toISOString()
          },
          {
            _id: '3',
            invoiceId: 'inv124',
            tipoOperacion: 'error',
            descripcion: 'Error al conectar con SIFEN',
            estado: 'error',
            fecha: new Date().toISOString()
          }
        ];
      } finally {
        loading.value = false;
      }
    };

    watch(currentPage, () => {
      updateUrl();
      loadLogs();
    });

    watch(itemsPerPage, () => {
      currentPage.value = 1;
      loadLogs();
    });

    watch(selectedFilter, () => {
      currentPage.value = 1;
      updateUrl();
      loadLogs();
    });

    onMounted(() => {
      loadLogs();
    });
    
    return {
      route,
      router,
      logs,
      loading,
      search,
      currentPage,
      totalPages,
      itemsPerPage,
      selectedFilter,
      filterOptions,
      headers,
      getLogStatusColor,
      getLogStateColor,
      formatDateTime,
      viewInvoice,
      changePage
    };
  }
};
</script>