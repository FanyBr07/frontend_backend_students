
<script setup>
import { ref, onMounted, computed } from 'vue'

const API = '/api/v2/roles'

const roles = ref([])
const busqueda = ref('')
const cargando = ref(true)
const error = ref('')
const mostrarFormulario = ref(false)
const editando = ref(false)
const guardando = ref(false)

const notificacion = ref({
  visible: false,
  mensaje: '',
  tipo: 'exito'
})

const rolPendienteEliminar = ref(null)

let temporizadorNotificacion

const formulario = ref({
  id: null,
  name: '',
  description: ''
})

// Filtrar roles por nombre
const rolesFiltrados = computed(() => {
  const texto = busqueda.value.trim().toLowerCase()

  if (!texto) return roles.value

  return roles.value.filter(rol =>
    (rol.name || '').toLowerCase().includes(texto)
  )
})

// Mostrar notificaciones integradas
function mostrarNotificacion(mensaje, tipo = 'exito') {
  notificacion.value = {
    visible: true,
    mensaje,
    tipo
  }

  clearTimeout(temporizadorNotificacion)

  temporizadorNotificacion = setTimeout(() => {
    notificacion.value.visible = false
  }, 3500)
}

// Consultar roles
async function cargarRoles() {
  cargando.value = true
  error.value = ''

  try {
    const respuesta = await fetch(API)

    if (!respuesta.ok) {
      throw new Error('No fue posible consultar los roles.')
    }

    const resultado = await respuesta.json()
    roles.value = resultado.data || []

  } catch (e) {
    error.value = e.message
    mostrarNotificacion(e.message, 'error')

  } finally {
    cargando.value = false
  }
}

// Preparar formulario para un nuevo rol
function nuevoRol() {
  editando.value = false

  formulario.value = {
    id: null,
    name: '',
    description: ''
  }

  mostrarFormulario.value = true
}

// Preparar formulario para editar
function editarRol(rol) {
  editando.value = true
  formulario.value = { ...rol }
  mostrarFormulario.value = true
}

// Crear o actualizar rol
async function guardarRol() {
  if (!formulario.value.name.trim()) return

  guardando.value = true
  error.value = ''

  try {
    const nombreRol = formulario.value.name.trim()
    const accion = editando.value ? 'actualizó' : 'registró'

    const url = editando.value
      ? `${API}/${formulario.value.id}`
      : API

    const respuesta = await fetch(url, {
      method: editando.value ? 'PUT' : 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        name: nombreRol,
        description: formulario.value.description.trim()
      })
    })

    if (!respuesta.ok) {
      throw new Error('No se pudo guardar el rol.')
    }

    mostrarFormulario.value = false

    await cargarRoles()

    mostrarNotificacion(
      `El rol "${nombreRol}" se ${accion} correctamente.`
    )

  } catch (e) {
    mostrarNotificacion(e.message, 'error')

  } finally {
    guardando.value = false
  }
}

// Solicitar confirmación antes de eliminar
function solicitarEliminacion(rol) {
  rolPendienteEliminar.value = rol
}

function cancelarEliminacion() {
  rolPendienteEliminar.value = null
}

// Eliminar rol
async function eliminarRol() {
  const rol = rolPendienteEliminar.value

  if (!rol) return

  try {
    const respuesta = await fetch(`${API}/${rol.id}`, {
      method: 'DELETE'
    })

    if (!respuesta.ok) {
      throw new Error('No se pudo eliminar el rol.')
    }

    rolPendienteEliminar.value = null

    await cargarRoles()

    mostrarNotificacion(
      `El rol "${rol.name}" se eliminó correctamente.`
    )

  } catch (e) {
    mostrarNotificacion(e.message, 'error')
  }
}

onMounted(cargarRoles)
</script>

<template>
  <div class="aplicacion">
    <header class="encabezado">
      <a class="marca" href="#">
        ARCHIVO<span>.</span>
      </a>

      <div class="encabezado-derecha">
        <span>SISTEMA DE GESTIÓN</span>
        <span class="version">VERSIÓN 01.0</span>
      </div>
    </header>

    <!-- Notificaciones -->
    <div
      v-if="notificacion.visible"
      class="notificacion"
      :class="notificacion.tipo"
      role="status"
      aria-live="polite"
    >
      <span>{{ notificacion.tipo === 'exito' ? '✓' : '!' }}</span>
      {{ notificacion.mensaje }}
    </div>

    <main class="contenido">
      <div class="ruta">
        <span>COLECCIÓN</span>
        <span>/</span>
        <span>ADMINISTRACIÓN</span>
        <span>/</span>
        <strong>ROLES</strong>
      </div>

      <section class="introduccion">
        <div>
          <p class="etiqueta">REGISTRO ADMINISTRATIVO · 001</p>
          <h1>Roles<span>.</span></h1>
          <p class="descripcion">
            Catálogo de permisos y niveles de acceso del sistema.
          </p>
        </div>

        <div class="numero-registros">
          <span>REGISTROS EN CATÁLOGO</span>
          <strong>{{ String(rolesFiltrados.length).padStart(3, '0') }}</strong>
        </div>
      </section>

      <section class="catalogo">
        <div class="catalogo-cabecera">
          <div>
            <span class="etiqueta">ÍNDICE GENERAL</span>
            <h2>Directorio de roles</h2>
          </div>

          <button class="boton-principal" @click="nuevoRol">
            <span>＋</span> Registrar rol
          </button>
        </div>

        <div v-if="error" class="mensaje-error">
          {{ error }}
          <button @click="cargarRoles">Reintentar</button>
        </div>

        <!-- Buscador -->
        <div class="busqueda-contenedor">
          <label for="buscar-rol">BUSCAR EN EL CATÁLOGO</label>
          <input
            id="buscar-rol"
            v-model="busqueda"
            type="search"
            placeholder="Escribe el nombre de un rol..."
          />
        </div>

        <div v-if="cargando" class="estado">
          Consultando el catálogo...
        </div>

        <div v-else-if="roles.length === 0" class="estado">
          No hay registros disponibles.
        </div>

        <div v-else-if="rolesFiltrados.length === 0" class="estado">
          No se encontraron roles con ese nombre.
        </div>

        <div v-else class="tabla-contenedor">
          <table>
            <thead>
              <tr>
                <th>REF.</th>
                <th>NOMBRE DEL ROL</th>
                <th>DESCRIPCIÓN</th>
                <th>ESTADO</th>
                <th>ACCIONES</th>
              </tr>
            </thead>

            <tbody>
              <tr
                v-for="(rol, indice) in rolesFiltrados"
                :key="rol.id"
              >
                <td class="referencia">
                  {{ String(indice + 1).padStart(3, '0') }}
                </td>

                <td class="nombre-rol">{{ rol.name }}</td>

                <td>{{ rol.description || 'Sin descripción' }}</td>

                <td>
                  <span class="estado-activo">● Registrado</span>
                </td>

                <td class="acciones">
                  <button title="Editar" @click="editarRol(rol)">
                    Editar
                  </button>

                  <button
                    title="Eliminar"
                    class="eliminar"
                    @click="solicitarEliminacion(rol)"
                  >
                    Eliminar
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Confirmación integrada -->
        <div
          v-if="rolPendienteEliminar"
          class="confirmacion-panel"
        >
          <div>
            <strong>Confirmar eliminación</strong>
            <p>
              ¿Deseas eliminar el rol
              <strong>{{ rolPendienteEliminar.name }}</strong>?
            </p>
          </div>

          <div class="confirmacion-acciones">
            <button
              class="boton-secundario"
              @click="cancelarEliminacion"
            >
              Conservar
            </button>

            <button
              class="boton-eliminar"
              @click="eliminarRol"
            >
              Sí, eliminar
            </button>
          </div>
        </div>

        <footer class="pie-tabla">
          <span>MOSTRANDO {{ rolesFiltrados.length }} DE {{ roles.length }} REGISTROS</span>
          <span>ARCHIVO / CATÁLOGO 001</span>
        </footer>
      </section>

      <!-- Formulario -->
      <section v-if="mostrarFormulario" class="formulario-panel">
        <div class="formulario-cabecera">
          <div>
            <span class="etiqueta">
              {{ editando ? 'MODIFICACIÓN DE REGISTRO' : 'NUEVO REGISTRO' }}
            </span>

            <h2>{{ editando ? 'Editar rol' : 'Registrar rol' }}</h2>
          </div>

          <button
            class="cerrar"
            @click="mostrarFormulario = false"
          >
            ×
          </button>
        </div>

        <form @submit.prevent="guardarRol">
          <label for="nombre">Nombre del rol</label>

          <input
            id="nombre"
            v-model="formulario.name"
            maxlength="30"
            placeholder="Ej. Supervisor"
            required
          />

          <label for="descripcion">Descripción</label>

          <textarea
            id="descripcion"
            v-model="formulario.description"
            placeholder="Describe las funciones de este rol..."
            rows="3"
          ></textarea>

          <div class="formulario-acciones">
            <button
              type="button"
              class="boton-secundario"
              @click="mostrarFormulario = false"
            >
              Cancelar
            </button>

            <button
              type="submit"
              class="boton-principal"
              :disabled="guardando"
            >
              {{ guardando ? 'Guardando...' : 'Guardar registro' }}
            </button>
          </div>
        </form>
      </section>
    </main>

    <footer class="pie-general">
      <span>ARCHIVO<span class="punto">.</span> SISTEMA DE GESTIÓN</span>
      <span>DESARROLLO DE SERVICIOS WEB · 2026</span>
    </footer>
  </div>
</template>
