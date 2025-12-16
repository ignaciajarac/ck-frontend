<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import axios from 'axios';

// --- ESTADO PRINCIPAL ---
const vista = ref('inicio');
const usuario = ref(null);
const esAdmin = computed(() => usuario.value && usuario.value.rol === 'admin');
const modoRegistro = ref(false);

// --- DATOS ---
const codigoQRReserva = ref('');
const misReservas = ref([]);
const menus = ref([]);
const tarifas = ref([]);
const extrasList = ref([]);
const reservasAdmin = ref([]);
const ticketsPendientesAdmin = ref([]);

// --- FORMULARIOS ---
const cred = ref({ correo: '', contraseña: '' });
const reg = ref({ nombre: '', apellido: '', correo: '', contraseña: '' });
const formReserva = ref({ fecha: '', bloque: '', ninos: 10, id_menu: null, tematica: '', extrasSeleccionados: [] });

// --- PAGO ---
const tarifaSeleccionada = ref(null);
const formPagoEntrada = ref({ numero_operacion: '', monto_transferido: null, fecha_transferencia: '' });

// --- INFO FIJA ---
const bloques = ["Mié-Jue: 15:00-17:00", "Mié-Jue: 18:00-20:00", "Vie-Sáb-Dom: 12:00-14:00", "Vie-Sáb-Dom: 15:00-17:00", "Vie-Sáb-Dom: 18:00-20:00"];
const datosBancarios = {
    nombre: 'Entretenimientos Roca Ltda.',
    rut: '77.817.491-K',
    banco: 'Banco de Chile',
    cuenta: 'Corriente N° 2202266001',
    correo_aviso: 'clubkidschillan@gmail.com'
};

// --- CARGA DE DATOS ---
const cargarDatos = async () => {
  try {
    const [m, t, e] = await Promise.all([
      axios.get('http://localhost:3000/api/menus'),
      axios.get('http://localhost:3000/api/tarifas'),
      axios.get('http://localhost:3000/api/extras')
    ]);
    menus.value = m.data; tarifas.value = t.data; extrasList.value = e.data;
  } catch (e) { console.error("Error conectando al servidor"); }
};

const cargarMisReservas = async (id_cliente) => {
    if (!id_cliente) return;
    try {
        const res = await axios.get(`http://localhost:3000/api/reservas/${id_cliente}`);
        misReservas.value = res.data;
    } catch (e) { 
        console.error("Error cargando reservas", e);
        misReservas.value = []; 
    }
};

watch(vista, (newVista) => {
    if (newVista === 'misreservas' && usuario.value) cargarMisReservas(usuario.value.id_cliente);
});

// --- AUTH ---
const login = async () => {
  try {
    const res = await axios.post('http://localhost:3000/api/auth/login', cred.value);
    usuario.value = res.data.usuario;
    if (esAdmin.value) cargarAdmin();
  } catch (e) { alert("❌ Datos incorrectos o error de servidor"); }
};

const registrarse = async () => {
  try {
    await axios.post('http://localhost:3000/api/auth/registro', reg.value);
    alert("✅ Cuenta creada. Inicia sesión.");
    modoRegistro.value = false;
  } catch (e) { alert("Error al registrar"); }
};

const salir = () => { 
    usuario.value = null; 
    vista.value = 'inicio'; 
    misReservas.value = [];
};

// --- ADMIN ---
const cargarAdmin = async () => {
    try {
        const res = await axios.get('http://localhost:3000/api/admin/reservas');
        reservasAdmin.value = res.data;
    } catch (e) { console.error("Error cargando admin"); }
};

// --- ENTRADAS ---
const iniciarCompraEntrada = (tarifa) => {
    if (!usuario.value) return alert("Inicia sesión primero.");
    tarifaSeleccionada.value = tarifa;
    formPagoEntrada.value.monto_transferido = tarifa.valor;
    vista.value = 'pagoentrada';
};

const enviarComprobante = async () => {
    if (!usuario.value) return alert("Error.");
    const qrCode = `CK-${Date.now().toString().slice(-6)}`;
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
    alert(`✅ Comprobante enviado. Estado: Pendiente.`);
    vista.value = 'inicio';
};

// --- CUMPLES ---
const reservarCumple = async () => {
  if (!usuario.value) return alert("Inicia sesión");
  const menu = menus.value.find(m => m.id_menu === formReserva.value.id_menu);
  if (!menu) return alert("Selecciona un menú");
  
  let total = menu.precio_por_nino * formReserva.value.ninos;
  // Sumar extras si los hay
  if (formReserva.value.extrasSeleccionados.length > 0) {
      formReserva.value.extrasSeleccionados.forEach(nombreExtra => {
          const extra = extrasList.value.find(e => e.nombre === nombreExtra);
          if (extra) total += extra.precio;
      });
  }
  
  try {
    await axios.post('http://localhost:3000/api/reservas', {
        id_cliente: usuario.value.id_cliente, 
        ...formReserva.value, 
        extras: formReserva.value.extrasSeleccionados, 
        total
    });
    
    codigoQRReserva.value = `RSV-${Date.now().toString().slice(-6)}`;
    await cargarMisReservas(usuario.value.id_cliente);
    vista.value = 'confirmacion';
    // Limpiar formulario
    formReserva.value = { fecha: '', bloque: '', ninos: 10, id_menu: null, tematica: '', extrasSeleccionados: [] };
  } catch (e) { alert("Error al crear reserva en servidor."); }
};

// --- ACCIONES ADMIN ---
const confirmarReserva = async (id_reserva) => {
    if (!confirm(`¿Confirmar reserva ID ${id_reserva}?`)) return;
    try {
        await axios.put(`http://localhost:3000/api/admin/reservas/confirmar/${id_reserva}`);
        alert("✅ Reserva Confirmada");
        cargarAdmin(); 
    } catch (e) { alert("Error al conectar con servidor"); }
};

const eliminarReserva = async (id_reserva) => {
    if (!confirm(`¿Eliminar reserva ID ${id_reserva}?`)) return;
    try {
        await axios.delete(`http://localhost:3000/api/admin/reservas/${id_reserva}`);
        alert("🗑️ Reserva Eliminada");
        reservasAdmin.value = reservasAdmin.value.filter(r => r.id_reserva !== id_reserva);
    } catch (e) { alert("Error al eliminar"); }
};

const confirmarEntrada = (id) => {
    if (!confirm("¿Activar QR?")) return;
    alert("✅ QR Activado");
    ticketsPendientesAdmin.value = ticketsPendientesAdmin.value.filter(t => t.id_comprobante !== id);
};

// --- HELPERS (Mejoras visuales) ---
const isUnlimited = (n) => n && n.toLowerCase().includes('ilimitado');
const getTicketDescription = (n) => {
    if (!n) return '';
    if (n.toLowerCase().includes('ilimitado')) return 'Acceso diario completo.';
    return 'Tiempo de juego asignado.';
};
const formatoPesos = (valor) => {
    return new Intl.NumberFormat('es-CL', { style: 'currency', currency: 'CLP' }).format(valor);
}
const generarCodigoReserva = (id) => {
    return `CK-RES-${id.toString().padStart(4, '0')}`;
}

onMounted(cargarDatos);
</script>

<template>
  <div class="app-container">
    
    <nav class="navbar glass">
      <img src="/logo.png" alt="ClubKids Logo" class="brand-logo">
      <div class="nav-links">
        <a @click="vista='inicio'" :class="{active: vista=='inicio'}">Inicio</a>
        <a @click="vista='entradas'" :class="{active: vista=='entradas'}">Entradas</a>
        <a @click="vista='infoCumples'" :class="{active: vista=='infoCumples'}">Info Cumples</a>
        <a @click="vista='cumples'" :class="{active: vista=='cumples'}">Reserva Cumple</a>
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
        <div class="hero-content">
          <div class="hero-spacer"></div>
          <div v-if="!usuario" class="hero-login-card">
            <div class="login-header"><h3>{{ modoRegistro ? 'Registro' : 'Bienvenido' }}</h3><p>{{ modoRegistro ? 'Crea tu cuenta' : 'Ingresa para reservar' }}</p></div>
            <div class="login-body">
              <div v-if="!modoRegistro">
                <input v-model="cred.correo" placeholder="Correo">
                <input v-model="cred.contraseña" type="password" placeholder="Contraseña">
                <button @click="login" class="btn-cta">Ingresar</button>
                <p class="switch" @click="modoRegistro=true">¿No tienes cuenta? <b>Regístrate</b></p>
              </div>
              <div v-else>
                <div class="row-inputs"><input v-model="reg.nombre" placeholder="Nombre"><input v-model="reg.apellido" placeholder="Apellido"></div>
                <input v-model="reg.correo" placeholder="Correo">
                <input v-model="reg.contraseña" type="password" placeholder="Contraseña">
                <button @click="registrarse" class="btn-cta">Crear Cuenta</button>
                <p class="switch" @click="modoRegistro=false">Volver</p>
              </div>
            </div>
          </div>
        </div>
      </header>
      <section class="precios-hero-section"><div class="precios-bg"></div></section>
      <section class="location-section"><div class="location-card-wide"><h3>📍 Ubicación</h3><p>Camino las mariposas nº 4009, Strip Center Parque Urbano.</p></div></section>
      <section class="gallery-section"><h2>Vive la Experiencia</h2><div class="gallery-grid"><div class="g-item" style="background-image: url('/trampolines.jpg');"><span>Trampolines</span></div><div class="g-item" style="background-image: url('/toboganes.jpg');"><span>Toboganes</span></div><div class="g-item" style="background-image: url('/laberinto.jpg');"><span>Laberinto</span></div><div class="g-item" style="background-image: url('/zonadebebes.jpg');"><span>Zona de Bebés</span></div><div class="g-item" style="background-image: url('/cafeteria.jpg');"><span>Cafetería</span></div><div class="g-item" style="background-image: url('/zonadecumpleaños.jpg');"><span>Zona de Cumpleaños</span></div></div></section>
      <footer class="modern-footer">
        <div class="footer-content">
            <div class="footer-col"><h3>🎈 ClubKids</h3><p>El mejor lugar para celebrar y jugar en Chillán.</p><div class="footer-address"><span>📍</span> Camino las mariposas nº 4009,<br>Strip Center Parque Urbano.</div></div>
            <div class="footer-col"><h3>⏰ Horarios de Atención</h3><ul class="footer-schedule"><li><span class="day">Lunes y Martes:</span> <span class="closed-tag">CERRADO</span></li><li><span class="day">Miércoles y Jueves:</span> 14:30 a 21:00 hrs</li><li><span class="day">Viernes a Domingo:</span> 12:00 a 21:00 hrs</li></ul></div>
            <div class="footer-col"><h3>📱 Contáctanos</h3><div class="social-links"><a href="#" class="social-item"><span>📸</span> @clubkidschillan</a><a href="#" class="social-item"><span>📘</span> ClubKids Chillán</a><a href="#" class="social-item"><span>📧</span> clubkidschillan@gmail.com</a><a href="#" class="social-item phone"><span>📞</span> +569 33 89 30 26</a></div></div>
        </div>
        <div class="footer-bottom"><p>© 2024 ClubKids Chillán. Todos los derechos reservados.</p></div>
      </footer>
    </main>

    <main v-if="vista === 'infoCumples'" class="info-cumples-layout">
      <div class="info-container">
        <h2>Todo sobre nuestros Cumpleaños</h2>
        <div class="info-slides-gallery"><div v-for="n in 10" :key="n" class="slide-item"><img :src="`/Informativo Cumpleaños${n}.jpeg`" :alt="'Info ' + n"></div></div>
        <button @click="vista='cumples'" class="btn-cta-large mt-4">¡Quiero Agendar mi Fiesta!</button>
      </div>
    </main>

    <main v-if="vista === 'entradas'" class="entradas-bg-container">
      <div class="tickets-grid-pop">
        <div v-for="(t, index) in tarifas" :key="t.id_tarifa" class="ticket-pop-card" :class="'pop-style-' + (index % 4)">
          <span class="deco-star">✨</span><h3>{{ t.tiempo }}</h3><div class="pop-price">{{ formatoPesos(t.valor) }}</div>
          <div class="ticket-details"><p class="time-desc">{{ getTicketDescription(t.tiempo) }}</p><ul class="inclusions-list"><li>Acceso a Trampolines y Toboganes.</li><li>Acceso a Laberinto y Zona de Bebés.</li><li>Acceso a Cafetería.</li><li v-if="isUnlimited(t.tiempo)" class="special-inclusion"><strong>Entrada y salida libre durante todo el día.</strong></li></ul></div>
          <button @click="iniciarCompraEntrada(t)" class="btn-pop">Comprar</button>
        </div>
      </div>
    </main>

    <main v-if="vista === 'pagoentrada'" class="pago-layout">
        <div class="pago-container">
            <h2 class="pago-title">Transferencia</h2>
            <div class="pago-content">
                <div class="banco-card">
                    <h3 class="banco-header">🏦 Datos</h3>
                    <div class="data-group"><span class="data-label">Beneficiario:</span><span class="data-value">{{ datosBancarios.nombre }}</span></div>
                    <div class="data-group"><span class="data-label">RUT:</span><span class="data-value">{{ datosBancarios.rut }}</span></div>
                    <div class="data-group"><span class="data-label">Banco:</span><span class="data-value">{{ datosBancarios.banco }}</span></div>
                    <div class="data-group"><span class="data-label">Cuenta:</span><span class="data-value">{{ datosBancarios.cuenta }}</span></div>
                    <div class="monto-final">Monto: <span class="monto-value">{{ formatoPesos(tarifaSeleccionada.valor) }}</span></div>
                </div>
                <form @submit.prevent="enviarComprobante" class="comprobante-form">
                    <h3 class="comprobante-header">📄 Comprobante</h3>
                    <div class="f-group"><label>Monto</label><input type="text" :value="formatoPesos(formPagoEntrada.monto_transferido)" disabled class="input-disabled"></div>
                    <div class="f-group"><label>N° Operación</label><input type="text" v-model="formPagoEntrada.numero_operacion" required></div>
                    <div class="f-group"><label>Fecha</label><input type="date" v-model="formPagoEntrada.fecha_transferencia" required></div>
                    <div class="pago-buttons"><button type="button" @click="vista='entradas'" class="btn-cancel">Cancelar</button><button type="submit" class="btn-pago-confirm">Enviar</button></div>
                </form>
            </div>
        </div>
    </main>

    <main v-if="vista === 'cumples'" class="cumples-bg-container">
      <div v-if="!usuario" class="no-auth-cumple-hero">
        <div class="no-auth-content"><h1 class="na-title">Reserva tu Fiesta</h1><div class="na-card"><span class="na-icon">🔒</span><h3>Acceso Restringido</h3><p>Inicia sesión para reservar.</p><button @click="vista='inicio'" class="btn-cta">Ir al Inicio</button></div></div>
      </div>
      <div v-else class="form-wrapper">
        <div class="header-party"><h2>🎂 Reserva tu Fiesta</h2></div>
        <form @submit.prevent="reservarCumple" class="form-party">
          <div class="party-section"><h3>📅 Datos</h3><div class="form-grid"><div class="f-group"><label>Fecha</label><input type="date" v-model="formReserva.fecha" required></div><div class="f-group"><label>Bloque</label><select v-model="formReserva.bloque" required><option v-for="b in bloques" :key="b" :value="b">{{ b }}</option></select></div><div class="f-group"><label>Niños</label><input type="number" v-model="formReserva.ninos" min="8" required></div><div class="f-group"><label>Temática</label><input v-model="formReserva.tematica" required></div></div></div>
          
          <div class="party-section">
            <h3>🎁 Selecciona tu Pack</h3>
            <div v-if="menus.length === 0" class="no-data-msg">Cargando menús disponibles...</div>
            <div class="pack-selector-pop">
              <div v-for="(m, index) in menus" :key="m.id_menu" class="pack-card-pop" :class="[{active: formReserva.id_menu === m.id_menu}, 'color-' + (index % 3)]" @click="formReserva.id_menu = m.id_menu">
                <div class="check-icon" v-if="formReserva.id_menu === m.id_menu">✔</div>
                <h4>{{ m.nombre }}</h4>
                <div class="pack-price-pop">{{ formatoPesos(m.precio_por_nino) }} <span>/ niño</span></div>
                <p class="pack-desc-mini">{{ m.descripcion }}</p>
              </div>
            </div>
          </div>

          <div class="party-section"><h3>✨ Extras</h3><div class="extras-grid-pop"><label v-for="ex in extrasList" :key="ex.id_extra" class="extra-bubble" :class="{selected: formReserva.extrasSeleccionados.includes(ex.nombre)}"><input type="checkbox" :value="ex.nombre" v-model="formReserva.extrasSeleccionados" hidden>{{ ex.nombre }} <span class="tag-price">+{{ formatoPesos(ex.precio) }}</span></label></div></div>
          <button class="btn-submit-party">📩 Confirmar Reserva</button>
        </form>
      </div>
    </main>

    <main v-if="vista === 'confirmacion'" class="confirmacion-layout">
        <div class="confirmacion-card"><div class="header-conf"><h1 class="conf-title">¡Reserva Confirmada!</h1></div><div class="conf-body"><div class="conf-qr-box"><span class="conf-qr-label">CÓDIGO:</span><span class="conf-qr-code">{{ codigoQRReserva }}</span></div></div><button class="btn-primary-lg btn-full" @click="vista='inicio'">Volver</button></div>
    </main>

    <main v-if="vista === 'misreservas'" class="historial-layout">
        <div class="historial-container">
            <h2>📜 Mis Reservas</h2>
            
            <div v-if="misReservas.length === 0" class="empty-state-card">
                <span class="empty-icon">📅</span>
                <h3>No tienes reservas activas</h3>
                <p>¿Qué esperas para celebrar con nosotros?</p>
                <button @click="vista='cumples'" class="btn-primary-sm mt-3">Reservar Ahora</button>
            </div>

            <div v-else class="reservas-grid">
                <div v-for="r in misReservas" :key="r.id_reserva" class="reserva-card-modern">
                    
                    <div class="card-header-modern">
                        <span class="status-badge-modern" :class="r.estado">{{ r.estado }}</span>
                        <span class="date-badge">🗓️ {{ r.fecha_evento.split('T')[0] }}</span>
                    </div>

                    <div class="card-body-modern">
                        <h3 class="pack-title">{{ r.menu_nombre }}</h3>
                        <div class="details-grid">
                            <p><strong>⏰ Bloque:</strong> {{ r.bloque_horario }}</p>
                            <p><strong>🎨 Temática:</strong> {{ r.tematica }}</p>
                            <p><strong>👶 Invitados:</strong> {{ r.cantidad_ninos }}</p>
                            <p v-if="r.extras"><strong>✨ Extras:</strong> {{ r.extras }}</p>
                        </div>
                    </div>

                    <div class="card-footer-modern">
                        <div class="code-box">
                            <small>CÓDIGO:</small>
                            <strong>{{ generarCodigoReserva(r.id_reserva) }}</strong>
                        </div>
                        <div class="total-box">
                            <small>Total Estimado</small>
                            <span class="total-amount">{{ formatoPesos(r.total_estimado) }}</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </main>

    <main v-if="vista === 'admin' && esAdmin" class="admin-layout">
      <div class="admin-container">
        <div class="admin-header"><h2>Panel Admin</h2><button @click="cargarAdmin">🔄</button></div>
        <h3 class="admin-section-title">Reservas</h3>
        <div class="table-wrapper mb-4">
          <table class="modern-table">
            <thead><tr><th>ID</th><th>Fecha</th><th>Cliente</th><th>Total</th><th>Estado</th><th>Acción</th></tr></thead>
            <tbody>
              <tr v-for="r in reservasAdmin" :key="r.id_reserva">
                <td>{{ r.id_reserva }}</td><td>{{ r.fecha_evento.split('T')[0] }}</td><td>{{ r.cliente }}</td><td>{{ formatoPesos(r.total_estimado) }}</td>
                <td><span class="r-status" :class="r.estado">{{ r.estado }}</span></td>
                <td class="admin-actions">
                    <button v-if="r.estado === 'Pendiente'" @click="confirmarReserva(r.id_reserva)" class="btn-confirm-admin btn-sm">Activar</button>
                    <button @click="eliminarReserva(r.id_reserva)" class="btn-delete-admin btn-sm">✕</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <hr>
        <h3 class="admin-section-title mt-4">Entradas Pendientes</h3>
        <div class="table-wrapper">
            <table class="modern-table">
                <thead><tr><th>Cliente</th><th>Monto</th><th>N° Op</th><th>Acción</th></tr></thead>
                <tbody>
                    <tr v-for="t in ticketsPendientesAdmin" :key="t.id_comprobante">
                        <td>{{ t.cliente }}</td><td>{{ formatoPesos(t.monto) }}</td><td>{{ t.num_operacion }}</td>
                        <td class="admin-actions"><button @click="confirmarEntrada(t.id_comprobante, t.sim_qr, t.correo)" class="btn-confirm-admin btn-sm">Activar</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
      </div>
    </main>

  </div>
</template>

<style>
/* GENERAL */
body { font-family: 'Segoe UI', sans-serif; background: #f3f4f6; margin: 0; color: #1f2937; }
.app-container { min-height: 100vh; display: flex; flex-direction: column; padding-top: 80px; }
.mt-3 { margin-top: 1rem; } .mt-4 { margin-top: 1.5rem; }

/* NAVBAR */
.navbar { display: flex; justify-content: space-between; align-items: center; padding: 15px 5%; background: white; position: fixed; width: 100%; top: 0; left: 0; z-index: 1000; box-shadow: 0 4px 10px rgba(0,0,0,0.05); height: 80px; box-sizing: border-box; }
.brand-logo { height: 50px; width: auto; }
.nav-links a { margin: 0 15px; cursor: pointer; color: #4b5563; font-weight: 600; text-decoration: none; }
.nav-links a.active { color: #ef4444; border-bottom: 2px solid #ef4444; }
.admin-tag { background: #fcd34d; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; }
.btn-primary-sm { background: #ef4444; color: white; border: none; padding: 8px 20px; border-radius: 50px; font-weight: bold; cursor: pointer; transition: 0.2s; }
.btn-primary-sm:hover { background: #d32f2f; }

/* HERO - LOGIN */
.hero-section { position: relative; height: calc(100vh - 80px); display: flex; align-items: center; justify-content: flex-end; background-color: #333; padding-right: 10%; margin-top: 0; padding-top: 0; }
.hero-bg { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-image: url('/banner.jpg'); background-size: cover; background-position: center; z-index: 0; }
.hero-overlay { display: none; } 
.hero-content { position: relative; z-index: 10; display: flex; width: 100%; justify-content: flex-end; }
.hero-login-card { background: white; width: 320px; border-radius: 15px; padding: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
.login-header h3 { margin: 0; color: #333; }
.login-header p { margin: 5px 0 15px 0; color: #666; font-size: 0.9rem; }
.login-body input { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; }
.row-inputs { display: flex; gap: 5px; }
.btn-cta { width: 100%; background: #ef4444; color: white; padding: 10px; border-radius: 8px; border: none; font-weight: bold; cursor: pointer; }
.switch { text-align: center; font-size: 0.8rem; margin-top: 10px; cursor: pointer; }

/* PRECIOS & UBICACION & GALERIA */
.precios-hero-section { position: relative; width: 100%; height: calc(100vh - 80px); overflow: hidden; background-color: #222; }
.precios-bg { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-image: url('/precios.jpg'); background-size: cover; background-position: center; }
.location-section { display: flex; justify-content: center; padding: 40px 5%; background-color: #f3f4f6; }
.location-card-wide { background: white; padding: 30px 50px; border-radius: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.08); text-align: center; border-left: 5px solid #ef4444; font-size: 1.2rem; color: #333; max-width: 800px; width: 100%; }
.gallery-section { padding: 40px 5%; text-align: center; }
.gallery-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); grid-auto-rows: 250px; gap: 20px; max-width: 1200px; margin: 30px auto; }
.g-item { border-radius: 15px; background-size: cover; background-position: center; position: relative; background-color: #ddd; overflow: hidden; transition: transform 0.3s; box-shadow: 0 5px 15px rgba(0,0,0,0.1); }
.g-item:hover { transform: scale(1.03); }
.g-item span { position: absolute; bottom: 0; left: 0; width: 100%; background: rgba(0,0,0,0.6); color: white; padding: 15px; font-weight: bold; font-size: 1.1rem; text-align: center; box-sizing: border-box; }

/* FOOTER */
.modern-footer { background-color: #2c3e50; color: #ecf0f1; padding: 50px 5% 20px; }
.footer-content { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 40px; max-width: 1200px; margin: 0 auto; }
.footer-col h3 { color: #f1c40f; margin-bottom: 20px; font-size: 1.3rem; border-bottom: 2px solid #e74c3c; padding-bottom: 10px; display: inline-block; }
.footer-col p { color: #bdc3c7; line-height: 1.6; }
.footer-address { margin-top: 15px; color: #ecf0f1; font-weight: 500; }
.footer-schedule { list-style: none; padding: 0; }
.footer-schedule li { margin-bottom: 10px; border-bottom: 1px dashed #34495e; padding-bottom: 5px; display: flex; justify-content: space-between; }
.closed-tag { background: #e74c3c; color: white; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; font-weight: bold; }
.social-links { display: flex; flex-direction: column; gap: 12px; }
.social-item { text-decoration: none; color: #ecf0f1; display: flex; align-items: center; gap: 10px; transition: 0.3s; }
.social-item:hover { color: #f1c40f; transform: translateX(5px); }
.social-item span { font-size: 1.2rem; }
.footer-bottom { text-align: center; margin-top: 40px; padding-top: 20px; border-top: 1px solid #34495e; font-size: 0.9rem; color: #7f8c8d; }

/* INFO CUMPLES */
.info-cumples-layout { padding: 40px 20px; background-color: #f8f9fa; min-height: 80vh; display: flex; justify-content: center; }
.info-container { max-width: 900px; width: 100%; text-align: center; }
.info-container h2 { color: #2c3e50; margin-bottom: 10px; font-size: 2.5rem; }
.info-intro { color: #666; margin-bottom: 30px; font-size: 1.1rem; }
.info-slides-gallery { display: flex; flex-direction: column; gap: 25px; margin-bottom: 30px; }
.slide-item img { width: 100%; height: auto; border-radius: 15px; box-shadow: 0 5px 20px rgba(0,0,0,0.1); display: block; }
.btn-cta-large { background: #ef4444; color: white; border: none; padding: 15px 40px; border-radius: 50px; font-weight: bold; font-size: 1.2rem; cursor: pointer; transition: background 0.3s, transform 0.2s; box-shadow: 0 5px 15px rgba(239, 68, 68, 0.4); }
.btn-cta-large:hover { background: #d32f2f; transform: translateY(-3px); }

/* ENTRADAS & CUMPLES FORM */
.entradas-bg-container { background-image: url('/fondo-entradas.jpg'); background-size: cover; background-position: center; min-height: 80vh; padding: 40px 20px; display: flex; justify-content: center; }
.tickets-grid-pop { display: flex; justify-content: center; gap: 25px; flex-wrap: wrap; align-items: flex-start; }
.ticket-pop-card { border-radius: 25px; padding: 30px 25px; width: 260px; text-align: center; background: white; box-shadow: 0 10px 25px rgba(0,0,0,0.2); position: relative; transition: transform 0.3s; display: flex; flex-direction: column; }
.ticket-pop-card:hover { transform: translateY(-10px) scale(1.02); }
.pop-price { font-size: 2.5rem; font-weight: 900; margin: 10px 0; color: #fff700; text-shadow: 0 2px 4px rgba(0,0,0,0.2); }
.ticket-details { text-align: left; margin: 15px 0 25px 0; color: rgba(255,255,255,0.95); font-size: 0.95rem; flex-grow: 1; }
.inclusions-list { list-style: none; padding: 0; margin: 0; }
.inclusions-list li { margin-bottom: 8px; padding-left: 25px; position: relative; line-height: 1.4; }
.inclusions-list li::before { content: '✓'; position: absolute; left: 0; top: 0; font-weight: bold; color: #bfff00; }
.btn-pop { background: #1a1a1a; color: white; border: none; padding: 12px 30px; border-radius: 50px; font-weight: bold; cursor: pointer; width: 100%; }
.pop-style-0 { background: linear-gradient(135deg, #00c6ff, #0072ff); color: white; }
.pop-style-1 { background: linear-gradient(135deg, #ff416c, #ff4b2b); color: white; }
.pop-style-2 { background: linear-gradient(135deg, #a855f7, #6366f1); color: white; }
.pop-style-3 { background: linear-gradient(135deg, #ff9966, #ff5e62); color: white; }
.cumples-bg-container { background-image: url('/fondo-cumple.jpg'); background-size: cover; background-position: center; min-height: 80vh; padding: 40px 20px; display: flex; justify-content: center; align-items: center; }
.no-auth-cumple-hero { width: 100%; max-width: 600px; text-align: center; color: white; }
.no-auth-content { background: rgba(0, 0, 0, 0.6); padding: 40px; border-radius: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
.na-title { font-size: 3rem; margin-bottom: 10px; font-weight: 800; text-shadow: 0 2px 5px rgba(0,0,0,0.5); }
.na-card { background: white; padding: 30px; border-radius: 15px; color: #333; max-width: 400px; margin: 0 auto; }
.form-wrapper { width: 100%; max-width: 800px; }
.form-party { background: white; padding: 40px; border-radius: 25px; }
.party-section { margin-bottom: 30px; }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 15px; }
.pack-selector-pop { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 15px; }
/* PACK CARDS MEJORADAS */
.pack-card-pop { border: 3px solid transparent; border-radius: 15px; padding: 25px; cursor: pointer; text-align: center; position: relative; color: white; transition: 0.3s; }
.pack-card-pop:hover { transform: translateY(-5px); box-shadow: 0 8px 20px rgba(0,0,0,0.2); }
.pack-card-pop.active { border-color: white; transform: scale(1.05); box-shadow: 0 10px 25px rgba(0,0,0,0.3); z-index: 2; }
.pack-desc-mini { font-size: 0.85rem; margin-top: 5px; opacity: 0.9; }
.color-0 { background: linear-gradient(135deg, #48dbfb, #0abde3); } .color-1 { background: linear-gradient(135deg, #ff9f43, #ee5253); } .color-2 { background: linear-gradient(135deg, #ff6b6b, #c0392b); }
.extras-grid-pop { display: flex; flex-wrap: wrap; gap: 10px; }
.extra-bubble { background: #f1f2f6; padding: 10px 20px; border-radius: 50px; cursor: pointer; }
.extra-bubble.selected { background: #5f27cd; color: white; }
.btn-submit-party { width: 100%; background: #ff4757; color: white; padding: 18px; border: none; border-radius: 15px; font-weight: bold; cursor: pointer; }

/* PAGO */
.pago-layout { display: flex; justify-content: center; padding: 40px 20px; min-height: 80vh; background-color: #f8f9fa; }
.pago-container { max-width: 800px; width: 100%; }
.pago-title { font-size: 2rem; border-bottom: 2px solid #eee; margin-bottom: 15px; }
.pago-content { display: flex; gap: 30px; flex-wrap: wrap; }
.banco-card { flex: 1; background: white; padding: 25px; border-radius: 15px; min-width: 300px; border-left: 4px solid #3498db; }
.banco-header { color: #3498db; border-bottom: 1px dashed #3498db; padding-bottom: 10px; }
.data-group { display: flex; justify-content: space-between; margin: 8px 0; }
.monto-final { background: #e8f5e9; padding: 10px; border-radius: 8px; margin-top: 15px; font-weight: bold; color: #27ae60; }
.comprobante-form { flex: 1; background: white; padding: 25px; border-radius: 15px; min-width: 300px; }
.comprobante-header { color: #ef4444; border-bottom: 1px dashed #fce4ec; padding-bottom: 10px; }
.pago-buttons { display: flex; gap: 10px; margin-top: 20px; }
.btn-cancel { flex: 1; background: #95a5a6; color: white; border: none; padding: 12px; border-radius: 8px; cursor: pointer; }
.btn-pago-confirm { flex: 1.5; background: #ef4444; color: white; border: none; padding: 12px; border-radius: 8px; cursor: pointer; font-weight: bold; }

/* CONFIRMACION */
.confirmacion-layout { min-height: 80vh; display: flex; justify-content: center; align-items: center; }
.confirmacion-card { background: white; padding: 50px; border-radius: 25px; text-align: center; border: 4px solid #2ecc71; max-width: 600px; }
.conf-qr-box { background: #e9f7ef; padding: 30px; border-radius: 15px; margin: 30px 0; border: 1px solid #c8e6c9; }
.conf-qr-code { font-size: 2rem; font-weight: bold; display: block; margin-top: 10px; }
.btn-primary-lg { background: #ef4444; color: white; border: none; padding: 15px 30px; border-radius: 10px; font-weight: bold; cursor: pointer; width: 100%; margin-top: 30px; }

/* === MIS RESERVAS (NUEVO ESTILO MODERNO) === */
.historial-layout { background-color: #f1f2f6; min-height: 80vh; padding: 40px 20px; display: flex; justify-content: center; }
.historial-container { width: 100%; max-width: 800px; }
.reservas-grid { display: flex; flex-direction: column; gap: 25px; }

/* Tarjeta de Reserva Moderna */
.reserva-card-modern {
    background: white; border-radius: 16px; overflow: hidden;
    box-shadow: 0 10px 30px rgba(0,0,0,0.08); transition: transform 0.2s;
    border: 1px solid #f0f0f0;
}
.reserva-card-modern:hover { transform: translateY(-3px); }

/* Cabecera de la tarjeta */
.card-header-modern {
    padding: 15px 25px; display: flex; justify-content: space-between; align-items: center;
    border-bottom: 1px solid #f0f0f0;
}
.status-badge-modern {
    padding: 6px 15px; border-radius: 30px; font-size: 0.85rem; font-weight: 800; text-transform: uppercase; letter-spacing: 1px;
}
.status-badge-modern.Pendiente { background: #fff3cd; color: #856404; }
.status-badge-modern.Confirmado { background: #d4edda; color: #155724; }
.date-badge { font-weight: 600; color: #555; font-size: 1.1rem; }

/* Cuerpo de la tarjeta */
.card-body-modern { padding: 25px; }
.pack-title { margin: 0 0 15px 0; color: #2c3e50; font-size: 1.4rem; }
.details-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 10px; }
.details-grid p { margin: 5px 0; color: #666; }

/* Pie de la tarjeta */
.card-footer-modern {
    padding: 20px 25px; background: #fafafa; border-top: 1px solid #f0f0f0;
    display: flex; justify-content: space-between; align-items: center;
}
.code-box small, .total-box small { display: block; color: #999; font-size: 0.8rem; margin-bottom: 5px; }
.code-box strong { font-family: monospace; font-size: 1.2rem; color: #333; background: #eee; padding: 5px 10px; border-radius: 5px; }
.total-amount { font-size: 1.5rem; font-weight: 900; color: #27ae60; }

/* Estado Vacío */
.empty-state-card {
    text-align: center; padding: 50px; background: white; border-radius: 20px; box-shadow: 0 5px 20px rgba(0,0,0,0.05);
}
.empty-icon { font-size: 4rem; display: block; margin-bottom: 20px; opacity: 0.5; }
.empty-state-card h3 { color: #333; margin-bottom: 10px; }
.empty-state-card p { color: #777; }
/* ========================================== */

/* ADMIN */
.admin-layout { background-color: #f1f2f6; min-height: 90vh; padding: 40px 20px; display: flex; justify-content: center; }
.admin-container { width: 100%; max-width: 1100px; background: white; border-radius: 15px; padding: 30px; }
.admin-header { display: flex; justify-content: space-between; margin-bottom: 30px; }
.modern-table { width: 100%; border-collapse: collapse; }
.modern-table th { background-color: #2c3e50; color: white; padding: 15px; text-align: left; }
.modern-table td { padding: 15px; border-bottom: 1px solid #eee; }
.btn-confirm-admin { background: #2ecc71; color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer; }
.btn-delete-admin { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 5px; cursor: pointer; margin-left: 10px; }
.r-status { background: #ffecb3; color: #ff8f00; padding: 3px 10px; border-radius: 20px; font-size: 0.8rem; font-weight: bold; }

/* RESPONSIVE */
@media (max-width: 768px) {
  .hero-section { justify-content: center; padding-right: 0; }
  .hero-bg { background-position: center top; } 
  .hero-login-card { width: 100%; margin: 20px; }
  .precios-hero-section { height: auto; min-height: 50vh; }
  .gallery-grid { grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); grid-auto-rows: 200px; }
  .ticket-pop-card { width: 100%; max-width: 300px; margin-bottom: 20px; }
  .footer-content { grid-template-columns: 1fr; }
  .card-header-modern, .card-footer-modern { flex-direction: column; align-items: flex-start; gap: 15px; }
  .total-box { align-self: flex-end; }
}
</style>