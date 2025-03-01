<template>
  <div style="background-color: white;">
    <div style="margin: 10px;">
      <!-- Contenedor para los filtros en línea -->
      <div style="display: flex; align-items: center; gap: 10px; flex-wrap: wrap;">
        <!-- Filtro Ficha -->
        <q-select
          v-model="idFicha"
          :options="fichasOptions"
          label="Selecciona una ficha"
          style="width: 300px;"
          emit-value
          option-label="label"
          option-value="value"
          map-options
        />

        <!-- Filtro Fecha -->
        <q-input
          v-model="fecha"
          label="Fecha"
          type="date"
          style="width: 200px;"
        />

        <!-- Botón estilo "Filtrar" -->
        <q-btn 
          @click="obtenerBitacorasFiltradas" 
          color="green-8" 
          style="border-radius: 8px; min-width: 120px; height: 40px;"
          icon="add"
        >
          <span>Filtrar</span>
        </q-btn>

        <!-- Botón de descarga PDF -->
        <q-btn 
          @click="generarPDF" 
          color="green-8" 
          style="border-radius: 8px; min-width: 120px; height: 40px;" 
          icon="file_download"
        >
          <span>DESCARGAR PDF</span>
        </q-btn>
      </div>

      <!-- Spinner de carga -->
      <div v-if="isLoading" class="q-pa-md">
        <q-spinner color="primary" size="50px" />
      </div>

      <!-- Tabla de registros -->
      <q-table
        v-if="!isLoading && rows.length > 0"
        title="Bitácoras"
        :rows="rows"
        :columns="columns"
        row-key="_id"
        flat
        bordered
        :dense="true"
      >
        <template v-slot:body-cell-estado="props">
          <q-td :props="props">
            {{ props.row.estado }}
          </q-td>
        </template>
      </q-table>

      <div v-if="!isLoading && rows.length === 0" class="q-pa-md">
        <p>No se encontraron bitácoras.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onBeforeMount } from 'vue';
import { useInformeStore } from '../stores/Informe.js'; // Store para bitácoras y fichas
import { Notify } from 'quasar';
import jsPDF from 'jspdf';
import autoTable from 'jspdf-autotable';

const informeStore = useInformeStore();
const isLoading = ref(false);
const rows = ref([]);
const idFicha = ref(''); // Aquí el valor será el _id de la ficha seleccionada
const fecha = ref('');

// Definir las columnas de la tabla
const columns = ref([
  { name: 'fecha', label: 'Fecha', field: 'fecha', align: 'left', sortable: true, format: val => new Date(val).toLocaleDateString() },
  { name: 'hora', label: 'Hora de Creación', field: 'createdAt', align: 'left', sortable: true, format: val => new Date(val).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }) },
  { name: 'nombre', label: 'Nombre Aprendiz', field: row => row.IdAprendis?.nombre || 'Desconocido', align: 'left', sortable: true },
  { name: 'cc', label: 'CC', field: row => row.IdAprendis?.cc || 'Desconocido', align: 'left', sortable: true },
  { name: 'IdFicha', label: 'IdFicha', field: row => row.IdAprendis?.IdFicha.nombre || 'No asignada', align: 'left', sortable: true },
  { name: 'estado', label: 'Estado', field: 'estado', align: 'left', sortable: true }
]);

// Opciones de fichas (serán cargadas desde la API)
const fichasOptions = ref([]);

// Cargar las fichas y todas las bitácoras al montar el componente
onBeforeMount(async () => {
  await cargarFichas();
  await obtenerTodasLasBitacoras(); // Cargar todas las bitácoras al iniciar
});

// Función para cargar todas las bitácoras al iniciar el componente
async function obtenerTodasLasBitacoras() {
  try {
    isLoading.value = true;
    const response = await informeStore.listarTodasLasBitacoras();

    if (Array.isArray(response) && response.length > 0) {
      rows.value = response;
    } else {
      rows.value = [];
      Notify.create({
        message: 'No se encontraron bitácoras.',
        color: 'negative',
      });
    }
  } catch (error) {
    console.error('Error al obtener todas las bitácoras:', error);
    Notify.create({
      message: 'Ocurrió un error al obtener las bitácoras.',
      color: 'negative',
    });
  } finally {
    isLoading.value = false;
  }
}

async function obtenerBitacorasFiltradas() {
  try {
    isLoading.value = true;

    if (!idFicha.value || !fecha.value) {
      Notify.create({
        message: 'Por favor, selecciona una ficha y una fecha.',
        color: 'negative',
      });
      isLoading.value = false;
      return;
    }

    // Convertir la fecha a ISO para la consulta
     const fechaISO =  new Date(fecha.value).toISOString().split('T')[0];
    /* const fechaISO = new Date(fecha.value).toISOString().split('T')[0]; */

    const response = await informeStore.listarBitacorasPorFichaYFecha(idFicha.value, fechaISO);

    if (Array.isArray(response) && response.length > 0) {
      rows.value = response;
    } else {
      rows.value = [];
      Notify.create({
        message: 'No se encontraron bitácoras para los filtros seleccionados.',
        color: 'negative',
      });
    }
  } catch (error) {
    console.error('Error al obtener las bitácoras:', error);
    Notify.create({
      message: 'Ocurrió un error al obtener las bitácoras.',
      color: 'negative',
    });
  } finally {
    isLoading.value = false;
  }
}

// Función para cargar las fichas
async function cargarFichas() {
  try {
    const fichas = await informeStore.obtenerFichas();

    // Verificar si fichas está definido y no es vacío
    if (fichas && fichas.length > 0) {
      fichasOptions.value = fichas.map(ficha => ({
        label: ficha.nombre, // Mostrar el nombre de la ficha
        value: ficha._id     // Guardar el ID de la ficha para usarlo en el backend
      }));
    } else {
      fichasOptions.value = [];
      Notify.create({
        message: 'No se encontraron fichas disponibles.',
        color: 'warning',
      });
    }
  } catch (error) {
    console.error('Error al cargar las fichas:', error);
    Notify.create({
      message: 'Error al cargar las fichas.',
      color: 'negative',
    });
  }
}
function generarPDF() {
  const doc = new jsPDF({
    orientation: 'landscape',
    unit: 'mm',
    format: 'a4'
  });

  // Tomar la fecha seleccionada en el filtro, sin convertirla a la fecha actual
  const fechaSeleccionada = new Date(fecha.value); // Usar fecha.value directamente
  const dia = fechaSeleccionada.getDate();
  const mes = fechaSeleccionada.toLocaleString('es-ES', { month: 'long' });
  const anio = fechaSeleccionada.getFullYear();

  // Generar la tabla con el título principal y el encabezado que incluye la fecha de entrada
  autoTable(doc, {
  startY: 10,
  head: [
    [
      { 
        content: `REGISTRO DE ASISTENCIA Y APROBACIÓN DEL ACTA No- DEL DÍA ${dia} DEL MES DE ${mes.toUpperCase()} DEL AÑO ${anio}`, 
        colSpan: 8, 
        styles: { halign: 'center', fontSize: 10, fontStyle: 'bold', fillColor: [255, 255, 255], textColor: [0, 0, 0] }
      }
    ],
    [
      { 
        content: 'OBJETIVO (S)', 
        colSpan: 8, // O ajusta el valor según sea necesario
        styles: { halign: 'left', fontSize: 10, fillColor: [255, 255, 255], textColor: [0, 0, 0] }
      }
    ], // Fila de "OBJETIVO (S)" alineada a la derecha
    ['No.', 'NOMBRES Y APELLIDOS', 'No. DOCUMENTO', 'PLANTA', 'CONTRATISTA', 'OTRO ¿CUÁL?', 'DEPENDENCIA/EMPRESA', 'CORREO ELECTRÓNICO', 'TELEFONO/EXT. SENA',
     'AUTORIZA GRABACION', 'FIRMA O PARTICIPACIÓN VIRTUAL']
  ],
  body: rows.value.map((row, index) => [
    index + 1,
    row.IdAprendis?.nombre || '',
    row.IdAprendis?.cc || '',
    '', '', '', 'SENA/CAT',
    row.IdAprendis?.email || '',
    row.IdAprendis?.telefono || '',
    '', ''
  ]),
  theme: 'grid',
  styles: { fontSize: 8, cellPadding: 3, lineWidth: 0.1, lineColor: [0, 0, 0] },
  headStyles: { fillColor: [255, 255, 255], textColor: [0, 0, 0], lineWidth: 0.1, lineColor: [0, 0, 0] },
  margin: { top: 10 },
});


  // Descargar el PDF
  doc.save('registro_asistencia.pdf');
}


</script>