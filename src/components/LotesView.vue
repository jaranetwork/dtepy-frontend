<template>
  <v-container fluid>
    <v-snackbar v-model="snackbar" :color="snackbarColor" :timeout="4000" location="top">
      <v-icon :start="snackbarIcon" size="large">{{ snackbarIcon }}</v-icon>
      {{ snackbarText }}
      <template v-slot:actions>
        <v-btn variant="text" @click="snackbar = false">Cerrar</v-btn>
      </template>
    </v-snackbar>

    <v-card>
      <v-card-title class="d-flex align-center flex-wrap">
        <h2>Lotes de Envío</h2>
        <v-spacer></v-spacer>
        <div class="d-flex align-center" style="gap: 8px;">
          <v-select
            v-model="filtroEstado"
            :items="estadosFiltro"
            label="Estado"
            single-line
            hide-details
            variant="outlined"
            density="compact"
            style="max-width: 160px;"
            clearable
          ></v-select>
          <v-btn color="warning" variant="outlined" @click="enviarPendientes" :loading="enviandoPendientes">
            <v-icon start>mdi-send-multiple</v-icon>
            Enviar Pendientes
          </v-btn>
          <v-btn color="info" variant="outlined" @click="consultarPendientes" :loading="consultandoPendientes">
            <v-icon start>mdi-refresh</v-icon>
            Consultar Pendientes
          </v-btn>
        </div>
      </v-card-title>

      <v-card-text>
          <v-data-table
            :headers="headers"
            :items="lotes"
            :loading="cargando"
            hide-default-footer
            item-value="_id"
          >
          <template v-slot:item.empresaId="{ item }">
            <span>{{ item.empresaId?.nombreFantasia || item.empresaId?.razonSocial || '-' }}</span>
          </template>

          <template v-slot:item.estado="{ item }">
            <v-chip :color="colorEstado(item.estado)" variant="flat" size="small">
              {{ textoEstado(item.estado) }}
            </v-chip>
          </template>

          <template v-slot:item.tipoDocumento="{ item }">
            <v-chip variant="outlined" size="small" color="primary">
              {{ item.descripcion || `Tipo ${item.tipoDocumento}` }}
            </v-chip>
          </template>

          <template v-slot:item.ambiente="{ item }">
            <v-chip
              :color="item.ambiente === 'produccion' ? 'error' : 'warning'"
              size="small" variant="outlined"
            >
              {{ item.ambiente?.toUpperCase() }}
            </v-chip>
          </template>

          <template v-slot:item.count="{ item }">
            <span class="font-weight-medium">{{ item.count }}/50</span>
          </template>

          <template v-slot:item.dProtConsLote="{ item }">
            <span class="text-mono text-caption">{{ item.dProtConsLote || '-' }}</span>
          </template>

          <template v-slot:item.createdAt="{ item }">
            {{ formatDate(item.createdAt) }}
          </template>

          <template v-slot:item.actions="{ item }">
            <v-menu v-model="item.menuOpen" :close-on-content-click="false" location="start">
              <template v-slot:activator="{ props }">
                <v-btn
                  color="primary"
                  size="small"
                  variant="tonal"
                  v-bind="props"
                  title="Acciones"
                >
                  <v-icon>mdi-dots-vertical</v-icon>
                </v-btn>
              </template>
              <v-list density="compact" min-width="200">
                <v-list-item @click="verDetalle(item); item.menuOpen = false">
                  <template v-slot:prepend>
                    <v-icon color="primary" size="small">mdi-eye</v-icon>
                  </template>
                  <v-list-item-title>Ver Detalle</v-list-item-title>
                </v-list-item>
                <v-list-item
                  v-if="item.estado === 'en_espera'"
                  @click="enviarLote(item); item.menuOpen = false"
                >
                  <template v-slot:prepend>
                    <v-icon color="warning" size="small">mdi-send</v-icon>
                  </template>
                  <v-list-item-title>Enviar</v-list-item-title>
                </v-list-item>
                <v-list-item
                  v-if="item.estado === 'procesando' || item.estado === 'enviado'"
                  @click="consultarLote(item); item.menuOpen = false"
                >
                  <template v-slot:prepend>
                    <v-icon color="info" size="small">mdi-refresh</v-icon>
                  </template>
                  <v-list-item-title>Consultar</v-list-item-title>
                </v-list-item>
                <v-divider
                  v-if="item.estado === 'en_espera' || item.estado === 'error'"
                ></v-divider>
                <v-list-item
                  v-if="item.estado === 'en_espera' || item.estado === 'error'"
                  @click="confirmarEliminar(item); item.menuOpen = false"
                >
                  <template v-slot:prepend>
                    <v-icon color="error" size="small">mdi-delete</v-icon>
                  </template>
                  <v-list-item-title class="text-error">Eliminar</v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>
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
          ></v-pagination>
        </div>
      </v-card-text>
    </v-card>

    <v-dialog v-model="dialogoEliminar" max-width="450" persistent>
      <v-card>
        <v-card-title class="text-h5 d-flex align-center bg-error text-white">
          <v-icon start>mdi-delete-forever</v-icon>
          Eliminar Lote
          <v-spacer></v-spacer>
          <v-btn
            icon="mdi-close"
            size="small"
            variant="text"
            @click="dialogoEliminar = false; loteSeleccionado = null"
            :disabled="eliminando"
            color="white"
          ></v-btn>
        </v-card-title>
        <v-card-text class="mt-4">
          <v-alert type="warning" variant="tonal" icon="mdi-alert" class="mb-3">
            <strong>⚠️ Advertencia:</strong> Esta acción eliminará el lote y desvinculará sus {{ loteSeleccionado?.count || 0 }} factura(s).
          </v-alert>
          <p class="text-body-1 mb-2">
            ¿Estás <strong>SEGURO</strong> de que deseas eliminar este lote?
          </p>
          <v-card variant="outlined" class="pa-3 mb-3">
            <div class="text-subtitle-1 font-weight-bold">{{ loteSeleccionado?.descripcion || `Lote #${loteSeleccionado?._id?.slice(-6)}` }}</div>
            <div class="text-caption text-medium-emphasis">
              <v-icon start size="x-small">mdi-counter</v-icon> {{ loteSeleccionado?.count || 0 }} factura(s)<br>
              <v-icon start size="x-small">mdi-domain</v-icon> {{ loteSeleccionado?.empresaId?.nombreFantasia || loteSeleccionado?.empresaId || '-' }}
            </div>
          </v-card>
          <p class="text-body-2 text-medium-emphasis">
            Esta acción <strong>no se puede deshacer</strong>.
          </p>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="grey"
            variant="text"
            @click="dialogoEliminar = false; loteSeleccionado = null"
            :disabled="eliminando"
          >
            Cancelar
          </v-btn>
          <v-btn
            color="error"
            variant="tonal"
            @click="eliminarLote"
            :loading="eliminando"
          >
            <v-icon start>mdi-delete-forever</v-icon>
            Eliminar
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import { ref, onMounted, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import axios from 'axios';

export default {
  name: 'LotesView',
  setup() {
    const route = useRoute();
    const router = useRouter();

    const lotes = ref([]);
    const cargando = ref(false);
    const filtroEstado = ref(null);
    const currentPage = ref(parseInt(route.query.page) || 1);
    const totalPages = ref(1);
    const itemsPerPage = ref(parseInt(route.query.limit) || 15);

    const enviandoLoteId = ref(null);
    const consultandoLoteId = ref(null);
    const enviandoPendientes = ref(false);
    const consultandoPendientes = ref(false);
    const dialogoEliminar = ref(false);
    const eliminando = ref(false);
    const loteSeleccionado = ref(null);

    const snackbar = ref(false);
    const snackbarText = ref('');
    const snackbarColor = ref('info');
    const snackbarIcon = ref('mdi-information');

    const estadosFiltro = [
      { title: 'En espera', value: 'en_espera' },
      { title: 'Enviado', value: 'enviado' },
      { title: 'Procesando', value: 'procesando' },
      { title: 'Completado', value: 'completado' },
      { title: 'Error', value: 'error' }
    ];

    const headers = [
      { title: 'Empresa', key: 'empresaId', sortable: true },
      { title: 'Tipo Documento', key: 'tipoDocumento' },
      { title: 'Ambiente', key: 'ambiente' },
      { title: 'Cantidad', key: 'count' },
      { title: 'Estado', key: 'estado' },
      { title: 'dProtConsLote', key: 'dProtConsLote' },
      { title: 'Creado', key: 'createdAt' },
      { title: 'Acciones', key: 'actions', sortable: false }
    ];

    const colorEstado = (estado) => {
      switch(estado) {
        case 'en_espera': return 'warning';
        case 'enviado': return 'info';
        case 'procesando': return 'primary';
        case 'completado': return 'success';
        case 'error': return 'error';
        default: return 'grey';
      }
    };

    const textoEstado = (estado) => {
      switch(estado) {
        case 'en_espera': return 'En espera';
        case 'enviado': return 'Enviado';
        case 'procesando': return 'Procesando';
        case 'completado': return 'Completado';
        case 'error': return 'Error';
        default: return estado;
      }
    };

    const formatDate = (dateString) => {
      if (!dateString) return '-';
      const d = new Date(dateString);
      return d.toLocaleString('es-PY');
    };

    const verDetalle = (item) => {
      router.push(`/lotes/${item._id}`);
    };

    const updateUrl = () => {
      const query = {};
      if (currentPage.value > 1) query.page = currentPage.value;
      if (itemsPerPage.value !== 15) query.limit = itemsPerPage.value;
      if (filtroEstado.value) query.estado = filtroEstado.value;
      router.replace({ query }).catch(() => {});
    };

    const cargarLotes = async () => {
      cargando.value = true;
      try {
        const params = new URLSearchParams();
        params.append('page', currentPage.value);
        params.append('limit', itemsPerPage.value);
        if (filtroEstado.value) params.append('estado', filtroEstado.value);
        const res = await axios.get(`/api/lotes/list?${params.toString()}`);
        lotes.value = res.data.data || [];
        totalPages.value = res.data.totalPages || 1;
      } catch (err) {
        console.error('Error cargando lotes:', err);
        snackbarText.value = '❌ Error cargando lotes';
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        cargando.value = false;
      }
    };

    watch(currentPage, () => {
      updateUrl();
      cargarLotes();
    });

    watch(itemsPerPage, () => {
      currentPage.value = 1;
      cargarLotes();
    });

    watch(filtroEstado, () => {
      currentPage.value = 1;
      updateUrl();
      cargarLotes();
    });

    const enviarLote = async (lote) => {
      enviandoLoteId.value = lote._id;
      try {
        const res = await axios.post(`/api/lotes/enviar/${lote._id}`);
        snackbarText.value = `✅ Lote enviado - dProtConsLote: ${res.data.data?.dProtConsLote || 'N/A'}`;
        snackbarColor.value = 'success';
        snackbarIcon.value = 'mdi-check-circle';
        snackbar.value = true;
        await cargarLotes();
      } catch (err) {
        snackbarText.value = `❌ Error: ${err.response?.data?.error || err.message}`;
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        enviandoLoteId.value = null;
      }
    };

    const consultarLote = async (lote) => {
      consultandoLoteId.value = lote._id;
      try {
        const res = await axios.post(`/api/lotes/consultar/${lote._id}`);
        if (res.data.data?.completado) {
          snackbarText.value = '✅ Lote completado';
          snackbarColor.value = 'success';
          snackbarIcon.value = 'mdi-check-circle';
        } else {
          snackbarText.value = `ℹ️ ${res.data.data?.mensaje || 'Aún en proceso'}`;
          snackbarColor.value = 'info';
          snackbarIcon.value = 'mdi-information';
        }
        snackbar.value = true;
        await cargarLotes();
      } catch (err) {
        snackbarText.value = `❌ Error: ${err.response?.data?.error || err.message}`;
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        consultandoLoteId.value = null;
      }
    };

    const enviarPendientes = async () => {
      enviandoPendientes.value = true;
      try {
        const res = await axios.post('/api/lotes/enviar-pendientes');
        const ok = res.data.data?.filter(r => r.success).length || 0;
        snackbarText.value = `✅ ${ok} lote(s) enviado(s)`;
        snackbarColor.value = 'success';
        snackbarIcon.value = 'mdi-check-circle';
        snackbar.value = true;
        await cargarLotes();
      } catch (err) {
        snackbarText.value = `❌ Error: ${err.message}`;
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        enviandoPendientes.value = false;
      }
    };

    const confirmarEliminar = (lote) => {
      loteSeleccionado.value = lote;
      dialogoEliminar.value = true;
    };

    const eliminarLote = async () => {
      if (!loteSeleccionado.value) return;
      eliminando.value = true;
      try {
        const res = await axios.delete(`/api/lotes/${loteSeleccionado.value._id}`);
        snackbarText.value = `✅ ${res.data.message}`;
        snackbarColor.value = 'success';
        snackbarIcon.value = 'mdi-check-circle';
        snackbar.value = true;
        dialogoEliminar.value = false;
        loteSeleccionado.value = null;
        await cargarLotes();
      } catch (err) {
        snackbarText.value = `❌ Error: ${err.response?.data?.error || err.message}`;
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        eliminando.value = false;
      }
    };

    const consultarPendientes = async () => {
      consultandoPendientes.value = true;
      try {
        await cargarLotes();
        const lotesPendientes = lotes.value.filter(l => l.estado === 'procesando' || l.estado === 'enviado');
        if (lotesPendientes.length === 0) {
          snackbarText.value = 'ℹ️ No hay lotes pendientes de consulta';
          snackbarColor.value = 'info';
          snackbarIcon.value = 'mdi-information';
          snackbar.value = true;
          return;
        }
        let completados = 0;
        for (const l of lotesPendientes) {
          try {
            const res = await axios.post(`/api/lotes/consultar/${l._id}`);
            if (res.data.data?.completado) completados++;
          } catch (e) { /* ignore individual errors */ }
        }
        snackbarText.value = `✅ ${completados}/${lotesPendientes.length} lote(s) completados`;
        snackbarColor.value = 'success';
        snackbarIcon.value = 'mdi-check-circle';
        snackbar.value = true;
        await cargarLotes();
      } catch (err) {
        snackbarText.value = `❌ Error: ${err.message}`;
        snackbarColor.value = 'error';
        snackbarIcon.value = 'mdi-alert-circle';
        snackbar.value = true;
      } finally {
        consultandoPendientes.value = false;
      }
    };

    onMounted(() => {
      if (route.query.estado) filtroEstado.value = route.query.estado;
      cargarLotes();
    });

    return {
      lotes, cargando, filtroEstado, estadosFiltro, headers,
      colorEstado, textoEstado, formatDate, verDetalle,
      enviarLote, consultarLote, enviarPendientes, consultarPendientes,
      enviandoLoteId, consultandoLoteId, enviandoPendientes, consultandoPendientes,
      dialogoEliminar, eliminando, loteSeleccionado,
      confirmarEliminar, eliminarLote,
      currentPage, totalPages, itemsPerPage,
      snackbar, snackbarText, snackbarColor, snackbarIcon
    };
  }
};
</script>

<style scoped>
.text-mono {
  font-family: 'Courier New', Courier, monospace;
}
</style>
