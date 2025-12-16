<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import axios from 'axios';

// --- ESTADO ---
const vista = ref('inicio');
const usuario = ref(null);
const esAdmin = computed(() => usuario.value && usuario.value.rol === 'admin');
const modoRegistro = ref(false);

// --- ESTADO DE COMPRA Y RESERVA ---
const codigoQRReserva = ref(''); 
const misReservas = ref([]); 
const menus = ref([]);
const tarifas = ref([]);
const extrasList = ref([]);
const reservasAdmin = ref([]);
const ticketsPendientesAdmin = ref([]); // NUEVO: Solicitudes de entrada pendientes

// --- FORMULARIOS ---
const cred = ref({ correo: '', contraseña: '' });
const reg = ref({ nombre: '', apellido: '', correo: '', contraseña: '' });
const formReserva = ref({ fecha: '', bloque: '', ninos: 10, id_menu: null, tematica: '', extrasSeleccionados: [] });

// --- FORMULARIO DE PAGO DE ENTRADA ---
const tarifaSeleccionada = ref(null);
const formPagoEntrada = ref({
    numero_operacion: '',
    monto_transferido: null,
    fecha_transferencia: ''
});

// --- INFO ---
const bloques = ["Mié-Jue: 15:00-17:00", "Mié-Jue: 18:00-20:00", "Vie-Sáb-Dom: 12:00-14:00", "Vie-Sáb-Dom: 15:00-17:00", "Vie-Sáb-Dom: 18:00-20:00"];

// --- DATOS BANCARIOS REALES (Según tu info consolidada) ---
const datosBancarios = {
    nombre: 'Entretenimientos Roca Ltda.',
    rut: '77.817.491-K',
    banco: 'Banco de Chile',
    cuenta: 'Corriente N° 2202266001',
    correo_aviso: 'clubkidschillan@gmail.com'
};

// --- CARGA ---
const cargarDatos = async () => {
  try {
    const [m, t, e] = await Promise.all([
      axios.get('http://localhost:3000/api/menus'),
      axios.get('http://localhost:3000/api/tarifas'),
      axios.get('http://localhost:3000/api/extras')
    ]);
    menus.value = m.data; tarifas.value = t.data; extrasList.value = e.data;
  } catch (e) { console.error("Error conexión backend"); }
};

// --- FUNCIÓN DE CARGA DE HISTORIAL POR CLIENTE ---
const cargarMisReservas = async (id_cliente) => {
    try {
        const res = await axios.get(`http://localhost:3000/api/reservas/${id_cliente}`);
        misReservas.value = res.data;
    } catch (e) {
        console.error("Error al cargar mis reservas:", e);
        misReservas.value = [];
    }
};

// --- WATCHER: Carga reservas cuando el usuario cambia de vista a 'misreservas' ---
watch(vista, (newVista) => {
    if (newVista === 'misreservas' && usuario.value) {
        cargarMisReservas(usuario.value.id_cliente);
    }
});


// --- AUTH ---
const login = async () => {
  try {
    const res = await axios.post('http://localhost:3000/api/auth/login', cred.value);
    usuario.value = res.data.usuario;
    if (esAdmin.value) cargarAdmin();
  } catch (e) { alert("❌ Datos incorrectos"); }
};

const registrarse = async () => {
  try {
    await axios.post('http://localhost:3000/api/auth/registro', reg.value);
    alert("✅ ¡Cuenta creada! Inicia sesión.");
    modoRegistro.value = false;
  } catch (e) { alert("Error al registrar"); }
};

const cargarAdmin = async () => {
    try {
        const res = await axios.get('http://localhost:3000/api/admin/reservas');
        reservasAdmin.value = res.data;
    } catch (e) {}
};

const salir = () => { usuario.value = null; vista.value = 'inicio'; };

// --- ACCIONES DE ENTRADA ---
const iniciarCompraEntrada = (tarifa) => {
    if (!usuario.value) return alert("Debes iniciar sesión para comprar entradas.");
    tarifaSeleccionada.value = tarifa;
    formPagoEntrada.value.monto_transferido = tarifa.valor; 
    vista.value = 'pagoentrada';
};

// 2. Enviar Comprobante (Simulación de POST y envío al panel Admin)
const enviarComprobante = async () => {
    if (!usuario.value || !tarifaSeleccionada.value) return alert("Error: No se seleccionó la tarifa.");
    
    if (!formPagoEntrada.value.numero_operacion || !formPagoEntrada.value.fecha_transferencia) {
        return alert("Por favor, completa el número de operación y la fecha.");
    }

    const qrCode = `CK-PEND-${Date.now().toString().slice(-6)}`;
    
    try {
        // En un sistema real, aquí se enviarían los datos a la BD con estado PENDIENTE_PAGO.
        // Simulamos la inserción de la venta:
        await axios.post('http://localhost:3000/api/entradas', {
            id_cliente: usuario.value.id_cliente, 
            id_tarifa: tarifaSeleccionada.value.id_tarifa, 
            valor: tarifaSeleccionada.value.valor,
        });

        // NUEVO: SIMULACIÓN DE REGISTRO EN PANEL ADMIN
        ticketsPendientesAdmin.value.push({
            id_comprobante: Date.now(),
            cliente: usuario.value.nombre + ' ' + usuario.value.apellido,
            correo: usuario.value.correo,
            tarifa: tarifaSeleccionada.value.tiempo,
            monto: formPagoEntrada.value.monto_transferido,
            num_operacion: formPagoEntrada.value.numero_operacion,
            fecha_trans: formPagoEntrada.value.fecha_transferencia,
            sim_qr: qrCode
        });
        
        alert(`✅ Comprobante Enviado.\n\nTu compra por $${tarifaSeleccionada.value.valor} ha quedado en estado PENDIENTE.\n\nEl administrador de ${datosBancarios.correo_aviso} validará tu transferencia para activar tu QR.\n\nCódigo QR PENDIENTE: [ ${qrCode} ]`);
        
        // Limpiar y volver
        tarifaSeleccionada.value = null;
        formPagoEntrada.value = { numero_operacion: '', monto_transferido: null, fecha_transferencia: '' };
        vista.value = 'inicio';

    } catch (e) {
        alert("❌ Error al procesar el comprobante.");
    }
};

const reservarCumple = async () => {
  if (!usuario.value) return alert("Inicia sesión");
  const menu = menus.value.find(m => m.id_menu === formReserva.value.id_menu);
  if (!menu) return alert("Selecciona menú");
  let total = menu.precio_por_nino * formReserva.value.ninos;
  try {
    await axios.post('http://localhost:3000/api/reservas', {
        id_cliente: usuario.value.id_cliente, ...formReserva.value, extras: formReserva.value.extrasSeleccionados, total
    });
    
    const qrcode = `RSV-${Date.now().toString().slice(-6)}-${Math.floor(Math.random() * 100)}`;
    codigoQRReserva.value = qrcode; 
    vista.value = 'confirmacion'; 
    
  } catch (e) { alert("Error reserva"); }
};

// --- FUNCIONES ADMINISTRATIVAS ---

// Confirmar Reserva de Cumpleaños
const confirmarReserva = async (id_reserva) => {
    if (!confirm(`¿Estás seguro de CONFIRMAR la reserva N° ${id_reserva}? Esto actualizará su estado a 'Confirmado'.`)) {
        return;
    }
    try {
        await axios.put(`http://localhost:3000/api/admin/reservas/confirmar/${id_reserva}`);
        alert(`✅ Reserva N° ${id_reserva} activada y confirmada.`);
        cargarAdmin(); // Recarga la tabla
    } catch (e) {
        alert("❌ Error al activar la reserva. Intenta de nuevo.");
    }
};

// NUEVO: Eliminar Reserva de Cumpleaños
const eliminarReserva = async (id_reserva) => {
    if (!confirm(`⚠️ ¿Estás seguro de ELIMINAR la reserva N° ${id_reserva}?\nEsta acción es irreversible.`)) {
        return;
    }
    try {
        // En un sistema real, aquí se llamaría a la ruta DELETE
        // Simulación: Eliminación de la lista local
        reservasAdmin.value = reservasAdmin.value.filter(r => r.id_reserva !== id_reserva);
        alert(`🗑️ Reserva N° ${id_reserva} eliminada correctamente.`);
        cargarAdmin(); // Recarga (para sincronizar con el backend si fuera necesario)
    } catch (e) {
        alert("❌ Error al intentar eliminar la reserva.");
    }
};


// NUEVO: Confirmar Comprobante de Entrada
const confirmarEntrada = (comprobanteId, sim_qr, clienteCorreo) => {
    if (!confirm(`¿Confirmar pago y ACTIVAR QR para el cliente con ID N° ${comprobanteId}?`)) {
        return;
    }
    // Lógica para enviar el QR al cliente (simulación)
    alert(`✅ ¡QR Activado!\nQR: ${sim_qr}\n\nNotificación enviada a ${clienteCorreo}.`);
    
    // Eliminar de la lista pendiente
    ticketsPendientesAdmin.value = ticketsPendientesAdmin.value.filter(t => t.id_comprobante !== comprobanteId);
};


onMounted(cargarDatos);
</script>

<template>
  <div class="app-container">
    
    <nav class="navbar glass">
      <img src="/logo.png" alt="ClubKids Logo" class="brand-logo">
      <div class="nav-links">
        <a @click="vista='inicio'" :class="{active: vista=='inicio'}">Inicio</a>
        <a @click="vista='entradas'" :class="{active: vista=='entradas'}">Entradas</a>
        <a @click="vista='cumples'" :class="{active: vista=='cumples'}">Cumpleaños</a>
        <a v-if="usuario && !esAdmin" @click="vista='misreservas'" :class="{active: vista=='misreservas'}">Mis Reservas</a>
        <a v-if="esAdmin" @click="vista='admin'" class="admin-tag">Admin</a>
      </div>
      <div class="auth-btn">
        <span v-if="usuario" class="user-welcome">Hola, {{ usuario.nombre }} <button @click="salir">✕</button></span>
        <button v-else @click="vista='inicio'" class="btn-primary-sm">Ingresar</button>
      </div>
    </nav>

    <main v-if="vista === 'inicio'">
      <header class="hero-section">
        <div class="hero-bg"></div>
        <div class="hero-overlay"></div>
        <div class="hero-content">
          <div class="hero-text">
            <h1>Diversión sin límites en <span class="highlight">Chillán</span></h1>
            <p>1000m² de camas elásticas, toboganes y aventuras.</p>
          </div>
          <div v-if="!usuario" class="hero-login-card">
            <div class="login-header">
              <h3>{{ modoRegistro ? 'Registro' : 'Bienvenido' }}</h3>
              <p>{{ modoRegistro ? 'Crea tu cuenta' : 'Ingresa para reservar' }}</p>
            </div>
            <div class="login-body">
              <div v-if="!modoRegistro">
                <input v-model="cred.correo" placeholder="Correo">
                <input v-model="cred.contraseña" type="password" placeholder="Contraseña">
                <button @click="login" class="btn-cta">Ingresar</button>
                <p class="switch" @click="modoRegistro=true">¿No tienes cuenta? <b>Regístrate</b></p>
              </div>
              <div v-else>
                <div class="row-inputs">
                   <input v-model="reg.nombre" placeholder="Nombre">
                   <input v-model="reg.apellido" placeholder="Apellido">
                </div>
                <input v-model="reg.correo" placeholder="Correo">
                <input v-model="reg.contraseña" type="password" placeholder="Contraseña">
                <button @click="registrarse" class="btn-cta">Crear Cuenta</button>
                <p class="switch" @click="modoRegistro=false">Volver</p>
              </div>
            </div>
          </div>
        </div>
      </header>

      <section class="features-section">
        <div class="feature-card"><h3>🎂 Cumpleaños</h3><p>Todo incluido.</p></div>
        <div class="feature-card"><h3>🎟️ Entradas</h3><p>Compra online.</p></div>
        <div class="feature-card"><h3>📍 Ubicación</h3><p>Parque Urbano.</p></div>
      </section>

      <section class="gallery-section">
        <h2>Vive la Experiencia</h2>
        <div class="gallery-grid">
          <div class="g-item large" style="background-image: url('/juegos.jpg');"><span>Trampolines</span></div>
          <div class="g-item" style="background-image: url('/cumple.jpg');"><span>Celebraciones</span></div>
          <div class="g-item" style="background-image: url('/banner.jpg');"><span>Toboganes</span></div>
        </div>
      </section>

      <section class="info-footer">
        <div class="info-col"><h4>Horarios</h4><p>Mié-Jue: 14:30-21:00</p><p>Finde: 12:00-21:00</p></div>
        <div class="info-col"><h4>Contacto</h4><p>+569 33 89 30 26</p></div>
      </section>
    </main>

    <main v-if="vista === 'entradas'" class="entradas-bg-container">
      <div class="tickets-grid-pop">
        <div v-for="(t, index) in tarifas" :key="t.id_tarifa" class="ticket-pop-card" :class="'pop-style-' + (index % 4)">
          <span class="deco-star">✨</span>
          <h3>{{ t.tiempo }}</h3><div class="pop-price">${{ t.valor }}</div>
          <button @click="iniciarCompraEntrada(t)" class="btn-pop">Comprar</button>
          <span class="deco-rocket">🚀</span>
        </div>
      </div>
    </main>

    <main v-if="vista === 'pagoentrada'" class="pago-layout">
        <div class="pago-container">
            <h2 class="pago-title">Transferencia Bancaria</h2>
            <p class="pago-subtitle">Completa tu pago de <strong>{{ tarifaSeleccionada.tiempo }} por ${{ tarifaSeleccionada.valor }}</strong> y registra el comprobante para la activación.</p>

            <div class="pago-content">
                
                <div class="banco-card">
                    <h3 class="banco-header">🏦 Datos de Transferencia</h3>
                    <div class="data-group"><span class="data-label">Beneficiario:</span><span class="data-value">{{ datosBancarios.nombre }}</span></div>
                    <div class="data-group"><span class="data-label">RUT:</span><span class="data-value">{{ datosBancarios.rut }}</span></div>
                    <div class="data-group"><span class="data-label">Banco:</span><span class="data-value">{{ datosBancarios.banco }}</span></div>
                    <div class="data-group"><span class="data-label">Cuenta:</span><span class="data-value">{{ datosBancarios.cuenta }}</span></div>
                    
                    <div class="monto-final">
                      Monto a Transferir: <span class="monto-value">${{ tarifaSeleccionada.valor }}</span>
                    </div>
                </div>

                <form @submit.prevent="enviarComprobante" class="comprobante-form">
                    <h3 class="comprobante-header">📄 Registrar Comprobante</h3>
                    <div class="f-group">
                        <label>Monto Transferido</label>
                        <input type="number" v-model="formPagoEntrada.monto_transferido" disabled class="input-disabled">
                    </div>
                    <div class="f-group">
                        <label>N° de Operación / Transferencia</label>
                        <input type="text" v-model="formPagoEntrada.numero_operacion" placeholder="Ej: 12345678" required>
                    </div>
                    <div class="f-group">
                        <label>Fecha de la Transferencia</label>
                        <input type="date" v-model="formPagoEntrada.fecha_transferencia" required>
                    </div>
                    
                    <p class="comprobante-info">⚠️ Una vez enviado, el administrador debe validar su pago manualmente. Recibirá su QR en el correo en breve.</p>
                    
                    <div class="pago-buttons">
                        <button type="button" @click="vista='entradas'" class="btn-cancel">Cancelar</button>
                        <button type="submit" class="btn-pago-confirm">Enviar Comprobante</button>
                    </div>
                </form>
            </div>
        </div>
    </main>

    <main v-if="vista === 'cumples'" class="cumples-bg-container">
      <div class="form-wrapper">
        <div class="header-party">
          <h2>🎂 Reserva tu Fiesta Soñada</h2>
          <p>Personaliza cada detalle y celebra en grande</p>
        </div>
        <div v-if="!usuario" class="login-alert-box">
          <h3>🔒 Acceso Restringido</h3>
          <p>Por favor inicia sesión para agendar tu evento.</p>
        </div>
        <form v-else @submit.prevent="reservarCumple" class="form-party">
          <div class="party-section">
            <h3>📅 Datos del Evento</h3>
            <div class="form-grid">
              <div class="f-group"><label>Fecha</label><input type="date" v-model="formReserva.fecha" required></div>
              <div class="f-group"><label>Bloque Horario</label><select v-model="formReserva.bloque" required><option v-for="b in bloques" :key="b" :value="b">{{ b }}</option></select></div>
              <div class="f-group"><label>Niños (Mín 8)</label><input type="number" v-model="formReserva.ninos" min="8" required></div>
              <div class="f-group"><label>Temática</label><input v-model="formReserva.tematica" placeholder="Ej: Mario, Barbie..." required></div>
            </div>
          </div>
          <div class="party-section">
            <h3>🎁 Elige tu Pack</h3>
            <div class="pack-selector-pop">
              <div v-for="(m, index) in menus" :key="m.id_menu" 
                   class="pack-card-pop" 
                   :class="[{active: formReserva.id_menu === m.id_menu}, 'color-' + (index % 3)]"
                   @click="formReserva.id_menu = m.id_menu">
                <div class="check-icon" v-if="formReserva.id_menu === m.id_menu">✔</div>
                <h4>{{ m.nombre }}</h4>
                <div class="pack-price-pop">${{ m.precio_por_nino }} <span>/ niño</span></div>
                <p>{{ m.descripcion }}</p>
              </div>
            </div>
          </div>
          <div class="party-section">
            <h3>✨ Extras y Adicionales</h3>
            <div class="extras-grid-pop">
              <label v-for="ex in extrasList" :key="ex.id_extra" 
                     class="extra-bubble" 
                     :class="{selected: formReserva.extrasSeleccionados.includes(ex.nombre)}">
                <input type="checkbox" :value="ex.nombre" v-model="formReserva.extrasSeleccionados" hidden>
                {{ ex.nombre }} <span class="tag-price">+${{ ex.precio }}</span>
              </label>
            </div>
          </div>
          <button class="btn-submit-party">📩 Confirmar Reserva</button>
        </form>
      </div>
    </main>

    <main v-if="vista === 'confirmacion'" class="confirmacion-layout">
        <div class="confirmacion-card">
            <div class="header-conf">
                <img src="/logo.png" alt="ClubKids Logo" class="conf-logo">
                <h1 class="conf-title">¡Reserva de Cumpleaños Confirmada!</h1>
                <p class="conf-subtitle">Tu solicitud ha sido registrada exitosamente. Hemos enviado los detalles y el comprobante a tu correo electrónico.</p>
            </div>
            
            <div class="conf-body">
                <div class="conf-qr-box">
                    <span class="conf-qr-label">TU CÓDIGO DE RESERVA:</span>
                    <span class="conf-qr-code">{{ codigoQRReserva }}</span>
                    <p>Guarda este código. Lo necesitarás para el abono y confirmación final.</p>
                </div>
                
                <p class="conf-final-message">
                    **Próximo Paso:** Nuestro equipo te contactará en las próximas 24 horas para verificar los detalles de tu evento y coordinar el pago de la reserva ($50.000).
                </p>
            </div>
            
            <button class="btn-primary-lg btn-full" @click="vista='inicio'">Volver al Inicio</button>
        </div>
    </main>

    <main v-if="vista === 'misreservas'" class="historial-layout">
        <div class="historial-container">
            <h2>📜 Mi Historial de Reservas</h2>
            <p v-if="!usuario" class="empty-message-user">Inicia sesión para ver tu historial de reservas.</p>
            <p v-else-if="misReservas.length === 0" class="empty-message">No tienes reservas de cumpleaños registradas aún. ¡Agenda la tuya!</p>

            <div v-else class="reservas-grid">
                <div v-for="r in misReservas" :key="r.id_reserva" class="reserva-card">
                    <div class="reserva-header">
                        <span class="r-status" :class="r.estado">{{ r.estado }}</span>
                        <h4>🎉 {{ r.fecha_evento.split('T')[0] }} ({{ r.bloque_horario }})</h4>
                    </div>
                    <div class="reserva-body">
                        <p><strong>Pack:</strong> {{ r.menu_nombre }}</p>
                        <p><strong>Temática:</strong> {{ r.tematica }}</p>
                        <p><strong>Invitados:</strong> {{ r.cantidad_ninos }} niños</p>
                        <p class="r-total">Total Estimado: <strong>${{ r.total_estimado }}</strong></p>
                    </div>
                </div>
            </div>
        </div>
    </main>

    <main v-if="vista === 'admin' && esAdmin" class="admin-layout">
      <div class="admin-container">
        
        <div class="admin-header">
          <div class="ah-text">
            <h2>🎛️ Panel de Control</h2>
            <p>Gestión de ClubKids</p>
          </div>
          <button @click="cargarAdmin" class="btn-refresh">🔄 Actualizar</button>
        </div>

        <h3 class="admin-section-title">🎂 1. Reservas de Cumpleaños</h3>
        <div class="table-wrapper mb-4">
          <table class="modern-table">
            <thead>
              <tr>
                <th>ID</th>
                <th>Fecha / Hora</th>
                <th>Cliente</th>
                <th>Evento / Temática</th>
                <th>Total</th>
                <th>Estado</th>
                <th>Acción</th> </tr>
            </thead>
            <tbody>
              <tr v-for="r in reservasAdmin" :key="r.id_reserva">
                <td>{{ r.id_reserva }}</td>
                <td>
                  <div class="td-date">
                    <span class="d-day">{{ r.fecha_evento.split('T')[0] }}</span>
                    <span class="d-time">{{ r.bloque_horario }}</span>
                  </div>
                </td>
                <td>
                  <div class="td-user">
                    <strong>{{ r.cliente }}</strong>
                    <small>{{ r.correo }}</small>
                  </div>
                </td>
                <td>
                  <div class="td-event">
                    <span class="badge-pack">{{ r.menu }}</span>
                    <small>Tema: {{ r.tematica }} ({{ r.cantidad_ninos }} niños)</small>
                  </div>
                </td>
                <td class="text-price">${{ r.total_estimado }}</td>
                <td>
                    <span class="r-status" :class="r.estado">{{ r.estado }}</span>
                </td>
                <td class="admin-actions">
                    <button 
                        v-if="r.estado === 'Pendiente'" 
                        @click="confirmarReserva(r.id_reserva)" 
                        class="btn-confirm-admin btn-sm">
                        Activar
                    </button>
                    <span v-else class="text-success">✅ Activo</span>
                    
                    <button @click="eliminarReserva(r.id_reserva)" class="btn-delete-admin btn-sm">
                        ✕ Eliminar
                    </button>
                </td>
              </tr>
              <tr v-if="reservasAdmin.length === 0">
                <td colspan="7" class="empty-row">No hay reservas registradas aún.</td>
              </tr>
            </tbody>
          </table>
        </div>
        
        <hr>

        <h3 class="admin-section-title mt-4">🎟️ 2. Comprobantes de Entrada Pendientes</h3>
        <div class="table-wrapper">
            <table class="modern-table">
                <thead>
                    <tr>
                        <th>ID Pago</th>
                        <th>Cliente</th>
                        <th>Tarifa</th>
                        <th>Monto</th>
                        <th>Operación / Fecha Transf.</th>
                        <th>Acción</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="t in ticketsPendientesAdmin" :key="t.id_comprobante">
                        <td>{{ t.id_comprobante.toString().slice(-4) }}</td>
                        <td>
                            <div class="td-user">
                                <strong>{{ t.cliente }}</strong>
                                <small>{{ t.correo }}</small>
                            </div>
                        </td>
                        <td>{{ t.tarifa }}</td>
                        <td class="text-price">${{ t.monto }}</td>
                        <td>
                            <div class="td-date">
                                <strong>N° Op: {{ t.num_operacion }}</strong>
                                <small>Fecha: {{ t.fecha_trans }}</small>
                            </div>
                        </td>
                        <td class="admin-actions">
                            <button 
                                @click="confirmarEntrada(t.id_comprobante, t.sim_qr, t.correo)" 
                                class="btn-confirm-admin btn-sm">
                                Activar QR
                            </button>
                        </td>
                    </tr>
                    <tr v-if="ticketsPendientesAdmin.length === 0">
                        <td colspan="6" class="empty-row-info">No hay comprobantes de entrada pendientes.</td>
                    </tr>
                </tbody>
            </table>
        </div>


      </div>
    </main>

  </div>
</template>

<style>
/* FUENTE Y GENERAL */
body { font-family: 'Segoe UI', sans-serif; background: #f3f4f6; margin: 0; color: #1f2937; }
.app-container { min-height: 100vh; display: flex; flex-direction: column; }

/* NAVBAR */
.navbar { display: flex; justify-content: space-between; align-items: center; padding: 15px 5%; background: white; position: sticky; top: 0; z-index: 1000; box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
.brand-logo { height: 50px; width: auto; }
.nav-links a { margin: 0 15px; cursor: pointer; color: #4b5563; font-weight: 600; }
.nav-links a.active { color: #ef4444; border-bottom: 2px solid #ef4444; }
.admin-tag { background: #fcd34d; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; }
.btn-primary-sm { background: #ef4444; color: white; border: none; padding: 8px 20px; border-radius: 50px; font-weight: bold; cursor: pointer; }

/* HERO */
.hero-section { position: relative; height: 500px; display: flex; align-items: center; justify-content: center; background-color: #333; }
.hero-bg { position: absolute; width: 100%; height: 100%; background-image: url('/banner.jpg'); background-size: cover; background-position: center; opacity: 0.6; }
.hero-content { position: relative; z-index: 10; display: flex; width: 90%; max-width: 1100px; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 40px; }
.hero-text { flex: 1; color: white; min-width: 300px; text-shadow: 0 2px 5px rgba(0,0,0,0.7); }
.hero-text h1 { font-size: 3rem; margin-bottom: 20px; font-weight: 800; }
.highlight { color: #fcd34d; }
.hero-login-card { background: white; width: 320px; border-radius: 15px; padding: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
input, select { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; font-family: inherit; }
.row-inputs { display: flex; gap: 5px; }
.btn-cta { width: 100%; background: #ef4444; color: white; padding: 10px; border-radius: 8px; border: none; font-weight: bold; cursor: pointer; }
.switch { text-align: center; font-size: 0.8rem; margin-top: 10px; cursor: pointer; }
.features-section { display: flex; justify-content: center; gap: 20px; padding: 40px 5%; margin-top: -30px; position: relative; z-index: 20; flex-wrap: wrap; }
.feature-card { background: white; padding: 20px; border-radius: 15px; width: 200px; box-shadow: 0 5px 15px rgba(0,0,0,0.05); text-align: center; }
.gallery-section { padding: 40px 5%; text-align: center; }
.gallery-grid { display: grid; grid-template-columns: 2fr 1fr; grid-template-rows: 200px 200px; gap: 15px; max-width: 900px; margin: 20px auto; }
.g-item { border-radius: 15px; background-size: cover; background-position: center; position: relative; background-color: #555; }
.g-item.large { grid-row: span 2; }
.g-item span { position: absolute; bottom: 10px; left: 10px; background: white; padding: 5px 15px; border-radius: 20px; font-weight: bold; font-size: 0.8rem; }
.info-footer { background: #1f2937; color: white; padding: 40px 5%; display: flex; justify-content: space-around; }

/* ENTRADAS */
.entradas-bg-container { background-image: url('/fondo-entradas.jpg'); background-size: cover; background-position: center; min-height: 80vh; display: flex; align-items: center; justify-content: center; padding: 40px 20px; background-color: #2c3e50; }
.tickets-grid-pop { display: flex; justify-content: center; gap: 25px; flex-wrap: wrap; }
.ticket-pop-card { border-radius: 25px; padding: 30px 25px; width: 220px; text-align: center; color: white; box-shadow: 0 10px 25px rgba(0,0,0,0.2); position: relative; overflow: hidden; transition: transform 0.3s; }
.ticket-pop-card:hover { transform: translateY(-10px) scale(1.02); }
.ticket-pop-card h3 { font-size: 1.8rem; margin-bottom: 10px; font-weight: 800; text-shadow: 0 2px 4px rgba(0,0,0,0.2); }
.pop-price { font-size: 2.5rem; font-weight: 900; margin: 15px 0; text-shadow: 0 2px 4px rgba(0,0,0,0.2); color: #fff700; }
.btn-pop { background: #1a1a1a; color: white; border: none; padding: 12px 30px; border-radius: 50px; font-weight: bold; cursor: pointer; box-shadow: 0 5px 15px rgba(0,0,0,0.3); transition: 0.2s; }
.btn-pop:hover { background: #000; transform: scale(1.05); }
.pop-style-0 { background: linear-gradient(135deg, #00c6ff, #0072ff); }
.pop-style-1 { background: linear-gradient(135deg, #ff416c, #ff4b2b); }
.pop-style-2 { background: linear-gradient(135deg, #a855f7, #6366f1); }
.pop-style-3 { background: linear-gradient(135deg, #ff9966, #ff5e62); }
.deco-star { position: absolute; top: 10px; left: 15px; font-size: 1.5rem; opacity: 0.5; }
.deco-rocket { position: absolute; bottom: 10px; right: 15px; font-size: 1.5rem; opacity: 0.5; transform: rotate(-45deg); }

/* VISTA PAGO ENTRADA */
.pago-layout { display: flex; justify-content: center; padding: 40px 20px; min-height: 80vh; background-color: #f8f9fa; }
.pago-container { max-width: 800px; width: 100%; }
.pago-title { color: #2c3e50; font-size: 2rem; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-bottom: 15px; }
.pago-subtitle { color: #7f8c8d; font-size: 1.1rem; margin-bottom: 30px; }
.pago-content { display: flex; gap: 30px; flex-wrap: wrap; }
.banco-card { flex: 1; background: white; padding: 25px; border-radius: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); min-width: 300px; border-left: 4px solid #3498db; }
.banco-header { color: #3498db; margin-top: 0; border-bottom: 1px dashed #3498db; padding-bottom: 10px; }
.data-group { display: flex; justify-content: space-between; margin: 8px 0; font-size: 0.95rem; }
.data-label { font-weight: 600; color: #576574; }
.data-value { color: #34495e; }
.monto-final { background: #e8f5e9; padding: 10px; border-radius: 8px; margin-top: 15px !important; font-size: 1rem !important; color: #27ae60 !important; font-weight: bold; }
.monto-value { color: #1a1a1a; font-size: 1.2rem; }
.comprobante-form { flex: 1; background: white; padding: 25px; border-radius: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); min-width: 300px; }
.comprobante-header { color: #ef4444; margin-top: 0; border-bottom: 1px dashed #fce4ec; padding-bottom: 10px; }
.comprobante-info { font-size: 0.8rem; color: #e67e22; margin-bottom: 20px; }
.pago-buttons { display: flex; justify-content: space-between; gap: 10px; margin-top: 20px; }
.btn-cancel { background: #95a5a6; color: white; border: none; padding: 12px; border-radius: 8px; font-weight: bold; cursor: pointer; flex: 1; }
.btn-pago-confirm { background: #ef4444; color: white; border: none; padding: 12px; border-radius: 8px; font-weight: bold; cursor: pointer; flex: 1.5; }


/* CUMPLES */
.cumples-bg-container { background-image: url('/fondo-cumple.jpg'); background-size: cover; background-position: center; min-height: 90vh; padding: 40px 20px; display: flex; justify-content: center; align-items: flex-start; background-color: #ff9f43; }
.form-wrapper { width: 100%; max-width: 800px; }
.header-party { text-align: center; color: white; margin-bottom: 30px; text-shadow: 0 2px 5px rgba(0,0,0,0.3); }
.header-party h2 { font-size: 2.5rem; margin-bottom: 5px; }
.form-party { background: white; padding: 40px; border-radius: 25px; box-shadow: 0 15px 40px rgba(0,0,0,0.2); }
.party-section { margin-bottom: 30px; }
.party-section h3 { color: #576574; border-bottom: 2px solid #f1f2f6; padding-bottom: 10px; margin-bottom: 15px; }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 15px; }
.f-group label { display: block; font-weight: 600; color: #576574; margin-bottom: 5px; }
.pack-selector-pop { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 15px; }
.pack-card-pop { border: 2px solid transparent; border-radius: 15px; padding: 20px; cursor: pointer; text-align: center; position: relative; transition: 0.2s; color: white; box-shadow: 0 5px 15px rgba(0,0,0,0.1); }
.pack-card-pop:hover { transform: translateY(-5px); }
.pack-card-pop.active { border: 4px solid white; transform: scale(1.02); box-shadow: 0 10px 25px rgba(0,0,0,0.2); }
.color-0 { background: #48dbfb; } .color-1 { background: #ff9f43; } .color-2 { background: #ff6b6b; }
.pack-price-pop { font-size: 1.5rem; font-weight: bold; margin: 10px 0; }
.pack-price-pop span { font-size: 0.9rem; font-weight: normal; opacity: 0.9; }
.check-icon { position: absolute; top: 10px; right: 10px; background: white; color: #2ecc71; width: 25px; height: 25px; border-radius: 50%; font-weight: bold; line-height: 25px; }
.extras-grid-pop { display: flex; flex-wrap: wrap; gap: 10px; }
.extra-bubble { background: #f1f2f6; padding: 10px 20px; border-radius: 50px; cursor: pointer; transition: 0.2s; border: 2px solid transparent; font-weight: 500; }
.extra-bubble:hover { background: #dfe4ea; }
.extra-bubble.selected { background: #5f27cd; color: white; border-color: #5f27cd; box-shadow: 0 4px 10px rgba(95, 39, 205, 0.3); }
.tag-price { font-size: 0.8rem; opacity: 0.7; margin-left: 5px; }
.btn-submit-party { width: 100%; background: #ff4757; color: white; padding: 18px; border: none; border-radius: 15px; font-size: 1.3rem; font-weight: bold; cursor: pointer; transition: 0.2s; margin-top: 10px; box-shadow: 0 5px 15px rgba(255, 71, 87, 0.4); }
.btn-submit-party:hover { background: #ff6b81; transform: translateY(-2px); }
.login-alert-box { background: white; padding: 40px; border-radius: 20px; text-align: center; max-width: 400px; margin: 0 auto; box-shadow: 0 10px 30px rgba(0,0,0,0.1); }

/* VISTA DE CONFIRMACIÓN */
.confirmacion-layout {
    min-height: 80vh; background: #f9fafb; display: flex; justify-content: center; align-items: center;
    padding: 60px 20px;
}
.confirmacion-card {
    background: white; padding: 50px; border-radius: 25px; max-width: 600px; width: 100%;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.1); text-align: center;
    border: 4px solid #2ecc71;
}
.header-conf { margin-bottom: 30px; }
.conf-logo { height: 80px; margin-bottom: 15px; }
.conf-title { color: #2ecc71; font-size: 2rem; margin-top: 0; }
.conf-subtitle { color: #576574; font-size: 1.1rem; }
.conf-qr-box {
    background: #e9f7ef; padding: 30px; border-radius: 15px; margin: 30px 0;
    border: 1px solid #c8e6c9;
}
.conf-qr-label {
    font-size: 0.9rem; font-weight: 600; color: #27ae60; display: block;
}
.conf-qr-code {
    font-size: 2rem; font-weight: 800; color: #34495e; display: block;
    margin-top: 10px;
}
.conf-body p { color: #7f8c8d; font-size: 0.9rem; }
.conf-final-message {
    padding: 15px; background: #fffde7; border-left: 5px solid #ffcc00;
    margin-top: 20px; text-align: left; font-style: italic;
}
.btn-primary-lg { 
    background: #ef4444; 
    color: white; 
    border: none; 
    padding: 15px 30px; 
    border-radius: 10px; 
    font-weight: bold; 
    cursor: pointer;
    font-size: 1.1rem;
}
.btn-full { width: 100%; margin-top: 30px; }


/* MIS RESERVAS */
.historial-layout {
    background-color: #f1f2f6; 
    min-height: 80vh; 
    padding: 40px 20px; 
    display: flex; 
    justify-content: center;
}
.historial-container {
    width: 100%; max-width: 900px;
}
.historial-container h2 {
    color: #34495e; 
    border-bottom: 2px solid #ddd; 
    padding-bottom: 10px; 
    margin-bottom: 30px;
}
.reservas-grid {
    display: grid; 
    grid-template-columns: repeat(auto-fit, minmax(350px, 1fr)); 
    gap: 20px;
}
.reserva-card {
    background: white;
    padding: 20px;
    border-radius: 15px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.05);
    transition: transform 0.2s;
}
.reserva-card:hover {
    transform: translateY(-5px);
}
.reserva-header {
    display: flex; 
    justify-content: space-between; 
    align-items: center; 
    border-bottom: 1px dashed #eee; 
    padding-bottom: 10px; 
    margin-bottom: 10px;
}
.reserva-header h4 {
    margin: 0; 
    font-weight: 600; 
    color: #1f2937;
}
.r-status {
    background: #ffecb3; 
    color: #ff8f00; 
    padding: 3px 10px; 
    border-radius: 20px; 
    font-size: 0.8rem; 
    font-weight: bold;
}
.r-status.Pendiente { background: #fce4ec; color: #d81b60; }
.r-status.Confirmado { background: #e8f5e9; color: #43a047; }
.reserva-body p {
    margin: 5px 0; 
    font-size: 0.95rem; 
    color: #4b5563;
}
.r-total {
    margin-top: 10px !important;
    font-size: 1.1rem !important;
    color: #2c3e50 !important;
}
.r-total strong {
    color: #27ae60;
}
.empty-message {
    text-align: center;
    padding: 50px;
    background: #fff;
    border-radius: 10px;
    color: #7f8c8d;
    font-style: italic;
}
.empty-message-user {
    text-align: center;
    padding: 50px;
    background: #fff3e0;
    border-radius: 10px;
    color: #e65100;
    font-weight: bold;
}


/* ADMIN */
.admin-layout { background-color: #f1f2f6; min-height: 90vh; padding: 40px 20px; display: flex; justify-content: center; }
.admin-container { width: 100%; max-width: 1100px; background: white; border-radius: 15px; box-shadow: 0 5px 20px rgba(0,0,0,0.05); overflow: hidden; padding: 30px; }
.admin-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; border-bottom: 2px solid #f1f2f6; padding-bottom: 20px; }
.ah-text h2 { margin: 0; color: #2c3e50; font-size: 1.8rem; }
.ah-text p { margin: 5px 0 0; color: #7f8c8d; }
.btn-refresh { background: #3498db; color: white; border: none; padding: 10px 20px; border-radius: 8px; cursor: pointer; transition: 0.2s; }
.btn-refresh:hover { background: #2980b9; }
.table-wrapper { overflow-x: auto; }
.modern-table { width: 100%; border-collapse: collapse; min-width: 800px; }
.modern-table thead th { background-color: #2c3e50; color: white; text-align: left; padding: 15px; font-weight: 600; letter-spacing: 0.5px; }
.modern-table tbody tr { border-bottom: 1px solid #eee; transition: background 0.2s; }
.modern-table tbody tr:hover { background-color: #f8f9fa; }
.modern-table td { padding: 15px; vertical-align: middle; color: #34495e; }
.td-date, .td-user, .td-event { display: flex; flex-direction: column; }
.d-day { font-weight: bold; color: #2c3e50; }
.d-time { font-size: 0.85rem; color: #7f8c8d; }
.td-user small { color: #95a5a6; }
.badge-pack { background: #e1f5fe; color: #0288d1; padding: 4px 10px; border-radius: 20px; font-size: 0.8rem; font-weight: bold; margin-bottom: 4px; display: inline-block; }
.text-center { text-align: center; font-weight: bold; }
.text-price { font-weight: bold; color: #27ae60; font-size: 1.1rem; }
.empty-row { text-align: center; padding: 40px; color: #95a5a6; font-style: italic; }
.empty-row-info { text-align: center; padding: 20px; color: #7f8c8d; font-style: italic; }

.admin-section-title { color: #2c3e50; margin-top: 30px; margin-bottom: 15px; border-bottom: 1px solid #ddd; padding-bottom: 5px; font-size: 1.4rem; }
.mb-4 { margin-bottom: 30px; }

.btn-confirm-admin {
    background: #2ecc71;
    color: white;
    border: none;
    padding: 8px 15px;
    border-radius: 5px;
    font-weight: bold;
    cursor: pointer;
    transition: 0.2s;
    font-size: 0.9rem;
}
.btn-confirm-admin:hover {
    background: #27ae60;
}
.btn-delete-admin {
    background: #e74c3c;
    color: white;
    border: none;
    padding: 8px 15px;
    border-radius: 5px;
    font-weight: bold;
    cursor: pointer;
    transition: 0.2s;
    font-size: 0.9rem;
    margin-left: 10px;
}
.btn-delete-admin:hover {
    background: #c0392b;
}
.admin-actions {
    min-width: 180px;
}
.btn-sm { padding: 6px 12px; font-size: 0.85rem; }
.text-success {
    color: #27ae60;
    font-weight: bold;
    font-size: 0.9rem;
}
</style>