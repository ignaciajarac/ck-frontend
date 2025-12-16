<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

// --- ESTADO ---
const vista = ref('inicio');
const usuario = ref(null);
const esAdmin = computed(() => usuario.value && usuario.value.rol === 'admin');
const modoRegistro = ref(false);

// --- DATOS ---
const menus = ref([]);
const tarifas = ref([]);
const extrasList = ref([]);
const reservasAdmin = ref([]);

// --- FORMULARIOS ---
const cred = ref({ correo: '', contraseña: '' });
const reg = ref({ nombre: '', apellido: '', correo: '', contraseña: '' });
const formReserva = ref({ fecha: '', bloque: '', ninos: 10, id_menu: null, tematica: '', extrasSeleccionados: [] });

// --- INFO ---
const bloques = ["Mié-Jue: 15:00-17:00", "Mié-Jue: 18:00-20:00", "Vie-Sáb-Dom: 12:00-14:00", "Vie-Sáb-Dom: 15:00-17:00", "Vie-Sáb-Dom: 18:00-20:00"];

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

// --- ACCIONES ---
const comprarEntrada = async (t) => {
  if (!usuario.value) return alert("Inicia sesión para comprar");
  if (!confirm(`¿Pagar $${t.valor} por ${t.tiempo}?`)) return;
  try {
    const res = await axios.post('http://localhost:3000/api/entradas', {
      id_cliente: usuario.value.id_cliente, id_tarifa: t.id_tarifa, valor: t.valor
    });
    alert(`🎟️ TICKET QR: [ ${res.data.qr} ]\nEnviado a ${usuario.value.correo}`);
  } catch (e) { alert("Error en la compra"); }
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
    alert(`🎂 SOLICITUD ENVIADA.\nTe contactaremos para el abono.\nTotal estimado: $${total}`);
    vista.value = 'inicio';
  } catch (e) { alert("Error reserva"); }
};

onMounted(cargarDatos);
</script>

<template>
  <div class="app-container">
    
    <nav class="navbar glass">
      <div class="brand"><span class="icon">🎈</span> ClubKids</div>
      <div class="nav-links">
        <a @click="vista='inicio'" :class="{active: vista=='inicio'}">Inicio</a>
        <a @click="vista='entradas'" :class="{active: vista=='entradas'}">Entradas</a>
        <a @click="vista='cumples'" :class="{active: vista=='cumples'}">Cumpleaños</a>
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

    <main v-if="vista === 'entradas'" class="view-content">
      <h2>Entradas Diarias</h2>
      <div class="tickets-grid">
        <div v-for="t in tarifas" :key="t.id_tarifa" class="ticket-card">
          <h3>{{ t.tiempo }}</h3><div class="price">${{ t.valor }}</div>
          <button @click="comprarEntrada(t)" class="btn-buy">Comprar</button>
        </div>
      </div>
    </main>

    <main v-if="vista === 'cumples'" class="view-content">
      <h2>Reserva tu Fiesta</h2>
      <div v-if="!usuario" class="login-alert">🔒 Inicia sesión para agendar.</div>
      <form v-else @submit.prevent="reservarCumple" class="form-modern">
        <div class="form-grid">
          <div class="f-group"><label>Fecha</label><input type="date" v-model="formReserva.fecha" required></div>
          <div class="f-group"><label>Bloque</label><select v-model="formReserva.bloque" required><option v-for="b in bloques" :key="b" :value="b">{{ b }}</option></select></div>
          <div class="f-group"><label>Niños</label><input type="number" v-model="formReserva.ninos" min="8" required></div>
          <div class="f-group"><label>Temática</label><input v-model="formReserva.tematica" required></div>
        </div>
        <h3>Elige Pack</h3>
        <div class="pack-selector">
          <div v-for="m in menus" :key="m.id_menu" class="pack-item" :class="{active: formReserva.id_menu === m.id_menu}" @click="formReserva.id_menu = m.id_menu">
            <div class="pack-head">{{ m.nombre }}</div><div class="pack-price">${{ m.precio_por_nino }}</div>
          </div>
        </div>
        <h3>Extras</h3>
        <div class="extras-grid">
          <label v-for="ex in extrasList" :key="ex.id_extra" class="extra-chip"><input type="checkbox" :value="ex.nombre" v-model="formReserva.extrasSeleccionados">{{ ex.nombre }}</label>
        </div>
        <button class="btn-submit">Solicitar</button>
      </form>
    </main>

    <main v-if="vista === 'admin' && esAdmin" class="view-content">
      <h2>Panel Admin</h2>
      <table class="admin-table">
        <thead><tr><th>Fecha</th><th>Cliente</th><th>Pack</th><th>Niños</th></tr></thead>
        <tbody>
          <tr v-for="r in reservasAdmin" :key="r.id_reserva">
            <td>{{ r.fecha_evento.split('T')[0] }}</td><td>{{ r.cliente }}</td><td>{{ r.menu }}</td><td>{{ r.cantidad_ninos }}</td>
          </tr>
        </tbody>
      </table>
    </main>

  </div>
</template>

<style>
/* FUENTE Y GENERAL */
body { font-family: 'Segoe UI', sans-serif; background: #f3f4f6; margin: 0; color: #1f2937; }
.app-container { min-height: 100vh; display: flex; flex-direction: column; }

/* NAVBAR */
.navbar { display: flex; justify-content: space-between; align-items: center; padding: 15px 5%; background: white; position: sticky; top: 0; z-index: 1000; box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
.brand { font-weight: 800; font-size: 1.5rem; color: #ef4444; }
.nav-links a { margin: 0 15px; cursor: pointer; color: #4b5563; font-weight: 600; }
.nav-links a.active { color: #ef4444; border-bottom: 2px solid #ef4444; }
.admin-tag { background: #fcd34d; padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; }
.btn-primary-sm { background: #ef4444; color: white; border: none; padding: 8px 20px; border-radius: 50px; font-weight: bold; cursor: pointer; }

/* HERO (SOLUCIÓN A IMAGEN ROTA) */
.hero-section { position: relative; height: 500px; display: flex; align-items: center; justify-content: center; background-color: #333; /* Color de respaldo */ }
.hero-bg { 
  position: absolute; width: 100%; height: 100%; 
  background-image: url('/banner.jpg'); /* FOTO LOCAL */
  background-size: cover; background-position: center; 
  opacity: 0.6; /* Un poco transparente para que se lea el texto */
}
.hero-content { position: relative; z-index: 10; display: flex; width: 90%; max-width: 1100px; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 40px; }
.hero-text { flex: 1; color: white; min-width: 300px; text-shadow: 0 2px 5px rgba(0,0,0,0.7); }
.hero-text h1 { font-size: 3rem; margin-bottom: 20px; font-weight: 800; }
.highlight { color: #fcd34d; }

/* LOGIN CARD */
.hero-login-card { background: white; width: 320px; border-radius: 15px; padding: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
input, select { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; }
.row-inputs { display: flex; gap: 5px; }
.btn-cta { width: 100%; background: #ef4444; color: white; padding: 10px; border-radius: 8px; border: none; font-weight: bold; cursor: pointer; }
.switch { text-align: center; font-size: 0.8rem; margin-top: 10px; cursor: pointer; }

/* FEATURES */
.features-section { display: flex; justify-content: center; gap: 20px; padding: 40px 5%; margin-top: -30px; position: relative; z-index: 20; flex-wrap: wrap; }
.feature-card { background: white; padding: 20px; border-radius: 15px; width: 200px; box-shadow: 0 5px 15px rgba(0,0,0,0.05); text-align: center; }

/* GALERIA (SOLUCIÓN) */
.gallery-section { padding: 40px 5%; text-align: center; }
.gallery-grid { display: grid; grid-template-columns: 2fr 1fr; grid-template-rows: 200px 200px; gap: 15px; max-width: 900px; margin: 20px auto; }
.g-item { 
  border-radius: 15px; background-size: cover; background-position: center; position: relative; 
  background-color: #555; /* COLOR RESPALDO POR SI FALLA LA FOTO */
}
.g-item.large { grid-row: span 2; }
.g-item span { position: absolute; bottom: 10px; left: 10px; background: white; padding: 5px 15px; border-radius: 20px; font-weight: bold; font-size: 0.8rem; }

/* FOOTER */
.info-footer { background: #1f2937; color: white; padding: 40px 5%; display: flex; justify-content: space-around; }

/* INTERIORES */
.view-content { padding: 40px 5%; max-width: 900px; margin: 0 auto; }
.tickets-grid { display: flex; gap: 20px; justify-content: center; flex-wrap: wrap; }
.ticket-card { background: white; padding: 20px; border-radius: 15px; width: 180px; text-align: center; border: 2px solid #eee; }
.ticket-card .price { font-size: 1.8rem; font-weight: bold; color: #ef4444; margin: 10px 0; }
.btn-buy { background: #1f2937; color: white; border: none; padding: 8px 20px; border-radius: 20px; cursor: pointer; }

/* FORM CUMPLES */
.form-modern { background: white; padding: 30px; border-radius: 15px; box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 15px; margin-bottom: 20px; }
.pack-selector { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; margin-bottom: 20px; }
.pack-item { border: 2px solid #eee; padding: 15px; border-radius: 10px; cursor: pointer; }
.pack-item.active { border-color: #ef4444; background: #fef2f2; }
.btn-submit { width: 100%; background: #ef4444; color: white; padding: 15px; border: none; border-radius: 10px; font-weight: bold; cursor: pointer; }
.login-alert { text-align: center; font-size: 1.2rem; color: #ef4444; margin-top: 50px; }
</style>