<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

// --- ESTADO ---
const vista = ref('inicio'); // inicio, entradas, cumples, admin
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

// --- INFO ESTÁTICA ---
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
  } catch (error) { console.error("Error cargando datos", error); }
};

// --- ACCIONES ---
const login = async () => {
  try {
    const res = await axios.post('http://localhost:3000/api/auth/login', cred.value);
    usuario.value = res.data.usuario;
    if (esAdmin.value) {
        const adminRes = await axios.get('http://localhost:3000/api/admin/reservas');
        reservasAdmin.value = adminRes.data;
    }
  } catch (e) { alert("❌ Datos incorrectos"); }
};

const registrarse = async () => {
  try {
    await axios.post('http://localhost:3000/api/auth/registro', reg.value);
    alert("✅ Registro exitoso. Ahora inicia sesión.");
    modoRegistro.value = false;
  } catch (e) { alert("Error al registrar (¿Correo repetido?)"); }
};

const salir = () => { usuario.value = null; vista.value = 'inicio'; };

// --- NEGOCIO (CON CORREO REAL) ---

const comprarEntrada = async (t) => {
  if (!usuario.value) return alert("Debes iniciar sesión para comprar");
  if (!confirm(`¿Confirmar pago de $${t.valor}?`)) return;

  try {
    // Enviamos correo y nombre para que el backend sepa a quién escribir
    await axios.post('http://localhost:3000/api/entradas', {
      id_cliente: usuario.value.id_cliente,
      id_tarifa: t.id_tarifa,
      valor: t.valor,
      correo_cliente: usuario.value.correo,
      nombre_cliente: usuario.value.nombre
    });
    
    // MENSAJE REAL
    alert(`✅ ¡Compra Exitosa!\n\nTe hemos enviado un correo a ${usuario.value.correo} con tu TICKET DIGITAL.\n\nPor favor revisa tu bandeja de entrada.`);
  } catch (e) { 
    alert("Error en la compra"); 
  }
};

const reservarCumple = async () => {
  if (!usuario.value) return alert("Inicia sesión");
  const menu = menus.value.find(m => m.id_menu === formReserva.value.id_menu);
  if (!menu) return alert("Selecciona menú");

  let total = menu.precio_por_nino * formReserva.value.ninos;

  try {
    await axios.post('http://localhost:3000/api/reservas', {
      id_cliente: usuario.value.id_cliente,
      ...formReserva.value,
      extras: formReserva.value.extrasSeleccionados,
      total,
      correo_cliente: usuario.value.correo,
      nombre_cliente: usuario.value.nombre
    });

    // MENSAJE REAL
    alert(`✅ ¡Solicitud Enviada!\n\nTe enviamos un correo a ${usuario.value.correo} con los datos para el abono.\n\nLa organizadora te contactará pronto.`);
    vista.value = 'inicio';
  } catch (e) { alert("Error al reservar"); }
};

onMounted(cargarDatos);
</script>

<template>
  <div class="app-container">
    <nav class="navbar">
      <div class="brand">🎈 ClubKids Chillán</div>
      <div class="nav-links">
        <a @click="vista='inicio'" :class="{active: vista=='inicio'}">Inicio</a>
        <a @click="vista='entradas'" :class="{active: vista=='entradas'}">Entradas</a>
        <a @click="vista='cumples'" :class="{active: vista=='cumples'}">Cumpleaños</a>
        <a v-if="esAdmin" @click="vista='admin'" class="admin-btn">Panel Admin</a>
      </div>
      <div class="auth-section">
        <span v-if="usuario">Hola, {{ usuario.nombre }} <button @click="salir">Salir</button></span>
        <button v-else @click="vista='inicio'" class="btn-login">Ingresar</button>
      </div>
    </nav>

    <main v-if="vista === 'inicio'">
      <div class="hero">
        <img src="https://images.unsplash.com/photo-1566737236500-c8ac43014a67?q=80&w=1920&auto=format&fit=crop" class="hero-bg">
        <div class="hero-overlay">
          <h1>¡Diversión sin límites en Chillán!</h1>
          <p>Más de 1000m² de juegos, trampolines y celebraciones inolvidables.</p>
        </div>
      </div>

      <div v-if="!usuario" class="login-wrapper">
        <div class="login-box">
          <h2>{{ modoRegistro ? 'Crear Cuenta' : 'Acceso Clientes' }}</h2>
          <div v-if="!modoRegistro">
            <input v-model="cred.correo" placeholder="Correo">
            <input v-model="cred.contraseña" type="password" placeholder="Contraseña">
            <button @click="login" class="btn-main">Entrar</button>
            <p @click="modoRegistro=true" class="switch-link">¿No tienes cuenta? Regístrate</p>
          </div>
          <div v-else>
            <input v-model="reg.nombre" placeholder="Nombre">
            <input v-model="reg.apellido" placeholder="Apellido">
            <input v-model="reg.correo" placeholder="Correo">
            <input v-model="reg.contraseña" type="password" placeholder="Contraseña">
            <button @click="registrarse" class="btn-sec">Registrarse</button>
            <p @click="modoRegistro=false" class="switch-link">Volver al Login</p>
          </div>
        </div>
      </div>

      <div class="info-grid">
        <div class="info-card">
          <h3>📍 Ubicación y Horarios</h3>
          <p><strong>Dirección:</strong> Alonso de Ercilla N°4009, Local K, Stripcenter Parque Urbano.</p>
          <hr>
          <p>🕒 <strong>Mié-Jue:</strong> 14:30 - 21:00</p>
          <p>🕒 <strong>Vie-Sáb-Dom:</strong> 12:00 - 21:00</p>
        </div>
        <div class="info-card rules">
          <h3>⚠️ Reglamento Importante</h3>
          <ul>
            <li>🧦 <strong>Calcetines Antideslizantes:</strong> OBLIGATORIO (Venta a $2.000).</li>
            <li>🚫 <strong>Alimentos:</strong> Prohibido ingresar comida externa.</li>
            <li>🤸 <strong>Seguridad:</strong> Prohibido volteretas mortales.</li>
            <li>👶 <strong>Menores de 4:</strong> Deben entrar acompañados.</li>
          </ul>
        </div>
      </div>
      
      <div class="bday-info-section">
        <h2>🎂 ¿Cómo funcionan los Cumpleaños?</h2>
        <div class="steps-container">
          <div class="step">
            <div class="step-num">1</div>
            <h4>Llegada</h4>
            <p>Entrega de pulseras y calcetines antideslizantes en recepción.</p>
          </div>
          <div class="step">
            <div class="step-num">2</div>
            <h4>Juego Libre</h4>
            <p>60 minutos de diversión en todas las instalaciones.</p>
          </div>
          <div class="step">
            <div class="step-num">3</div>
            <h4>Canto y Torta</h4>
            <p>Reunión en la mesa decorada para cantar y comer.</p>
          </div>
          <div class="step">
            <div class="step-num">4</div>
            <h4>Cierre</h4>
            <p>Apertura de regalos y despedida (Hora exacta).</p>
          </div>
        </div>
      </div>
    </main>

    <main v-if="vista === 'entradas'" class="tickets-view">
      <div class="tickets-header">
        <h2>🎟️ Compra de Entradas</h2>
        <p>Selecciona tu tiempo de juego. Recibirás tu QR al correo.</p>
      </div>
      <div class="tickets-grid">
        <div v-for="t in tarifas" :key="t.id_tarifa" class="ticket-card">
          <h3>{{ t.tiempo }}</h3>
          <div class="price">${{ t.valor }}</div>
          <button @click="comprarEntrada(t)" class="btn-buy">Comprar Ticket</button>
        </div>
      </div>
    </main>

    <main v-if="vista === 'cumples'" class="bday-view">
      <div class="bday-header">
        <img src="https://images.unsplash.com/photo-1530103862676-de3c9da59af7?q=80&w=1920&auto=format&fit=crop" class="bday-bg">
        <div class="overlay">
          <h2>Reserva tu Celebración</h2>
          <p>Elige temática, menú y fecha. Nosotros nos encargamos del resto.</p>
        </div>
      </div>

      <div v-if="!usuario" class="login-alert">⚠️ Por favor inicia sesión para agendar.</div>

      <div v-else class="form-container">
        <form @submit.prevent="reservarCumple">
          <div class="form-group">
            <h3>📅 1. Fecha y Hora</h3>
            <div class="row">
              <input type="date" v-model="formReserva.fecha" required>
              <select v-model="formReserva.bloque" required>
                <option value="">Selecciona Horario</option>
                <option v-for="b in bloques" :key="b" :value="b">{{ b }}</option>
              </select>
            </div>
          </div>

          <div class="form-group">
            <h3>🎉 2. Detalles de la Fiesta</h3>
            <div class="row">
              <input type="number" v-model="formReserva.ninos" min="8" placeholder="Cant. Niños (Mín 8)" required>
              <input v-model="formReserva.tematica" placeholder="Temática (Ej: Sonic, Barbie...)" required>
            </div>
          </div>

          <div class="form-group">
            <h3>🍔 3. Elige el Pack</h3>
            <div class="menus-list">
              <div v-for="m in menus" :key="m.id_menu" 
                   class="menu-item" 
                   :class="{selected: formReserva.id_menu === m.id_menu}"
                   @click="formReserva.id_menu = m.id_menu">
                <div class="m-head">
                  <strong>{{ m.nombre }}</strong>
                  <span>${{ m.precio_por_nino }}/niño</span>
                </div>
                <p>{{ m.descripcion }}</p>
              </div>
            </div>
          </div>

          <div class="form-group">
            <h3>➕ 4. Extras (Opcional)</h3>
            <div class="extras-list">
              <label v-for="ex in extrasList" :key="ex.id_extra" class="extra-item">
                <input type="checkbox" :value="ex.nombre" v-model="formReserva.extrasSeleccionados">
                {{ ex.nombre }} (${{ ex.precio }})
              </label>
            </div>
          </div>

          <button type="submit" class="btn-reserve">Solicitar Reserva</button>
          <p class="disclaimer">Al enviar, recibirás un correo con los datos de transferencia.</p>
        </form>
      </div>
    </main>

    <main v-if="vista === 'admin' && esAdmin" class="admin-view">
      <h2>Panel de Administración</h2>
      <div class="table-responsive">
        <table>
          <thead>
            <tr><th>Fecha</th><th>Cliente</th><th>Menú</th><th>Niños</th><th>Temática</th><th>Total</th></tr>
          </thead>
          <tbody>
            <tr v-for="r in reservasAdmin" :key="r.id_reserva">
              <td>{{ r.fecha_evento.split('T')[0] }}<br><small>{{ r.bloque_horario }}</small></td>
              <td>{{ r.cliente }}<br><small>{{ r.correo }}</small></td>
              <td>{{ r.menu }}</td>
              <td>{{ r.cantidad_ninos }}</td>
              <td>{{ r.tematica }}</td>
              <td>${{ r.total_estimado }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </main>

  </div>
</template>

<style scoped>
/* GENERAL */
.app-container { font-family: 'Segoe UI', sans-serif; background: #f8f9fa; min-height: 100vh; color: #333; }
.navbar { background: #fff; padding: 15px 30px; display: flex; justify-content: space-between; align-items: center; border-bottom: 3px solid #ff6b6b; position: sticky; top: 0; z-index: 100; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
.brand { font-size: 1.5rem; font-weight: bold; color: #ff6b6b; }
.nav-links a { margin: 0 15px; cursor: pointer; color: #555; font-weight: 500; transition: 0.2s; }
.nav-links a.active { color: #3498db; border-bottom: 2px solid #3498db; }
.btn-login { background: #3498db; color: white; border: none; padding: 8px 20px; border-radius: 20px; cursor: pointer; }
.admin-btn { color: #f1c40f !important; font-weight: bold; }

/* HERO */
.hero { position: relative; height: 300px; overflow: hidden; background: #333; }
.hero-bg { width: 100%; height: 100%; object-fit: cover; opacity: 0.6; }
.hero-overlay { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); text-align: center; color: white; width: 80%; }
.hero-overlay h1 { font-size: 2.5rem; margin-bottom: 10px; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }

/* LOGIN */
.login-wrapper { display: flex; justify-content: center; margin-top: -50px; position: relative; z-index: 10; padding: 20px; }
.login-box { background: white; padding: 30px; border-radius: 10px; box-shadow: 0 5px 15px rgba(0,0,0,0.1); width: 300px; text-align: center; }
.login-box input { width: 100%; padding: 10px; margin: 5px 0; border: 1px solid #ddd; border-radius: 5px; box-sizing: border-box; }
.btn-main { width: 100%; background: #ff6b6b; color: white; border: none; padding: 10px; border-radius: 5px; cursor: pointer; font-weight: bold; margin-top: 10px; }
.btn-sec { width: 100%; background: #3498db; color: white; border: none; padding: 10px; border-radius: 5px; cursor: pointer; font-weight: bold; margin-top: 10px; }
.switch-link { color: #3498db; cursor: pointer; font-size: 0.9rem; margin-top: 10px; text-decoration: underline; }

/* INFO GRID */
.info-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; padding: 40px; max-width: 1000px; margin: 0 auto; }
.info-card { background: white; padding: 25px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
.rules { border-left: 5px solid #ff9f43; }
.rules ul { padding-left: 20px; }
.rules li { margin-bottom: 8px; }

/* STEPS */
.bday-info-section { background: white; padding: 40px; text-align: center; margin-top: 20px; }
.steps-container { display: flex; flex-wrap: wrap; justify-content: center; gap: 30px; margin-top: 30px; }
.step { width: 200px; }
.step-num { width: 40px; height: 40px; background: #3498db; color: white; border-radius: 50%; line-height: 40px; margin: 0 auto 10px; font-weight: bold; }

/* TICKETS */
.tickets-view { padding: 40px; text-align: center; }
.tickets-grid { display: flex; flex-wrap: wrap; justify-content: center; gap: 20px; margin-top: 30px; }
.ticket-card { background: white; width: 220px; padding: 20px; border-radius: 10px; border: 2px solid #eee; transition: 0.3s; }
.ticket-card:hover { border-color: #3498db; transform: translateY(-5px); }
.ticket-card .price { font-size: 2rem; color: #3498db; font-weight: bold; margin: 15px 0; }
.btn-buy { background: #2ecc71; color: white; border: none; padding: 10px 20px; border-radius: 20px; cursor: pointer; font-weight: bold; }

/* BDAY FORM */
.bday-header { position: relative; height: 200px; overflow: hidden; background: #2c3e50; color: white; display: flex; align-items: center; justify-content: center; text-align: center; }
.bday-bg { position: absolute; width: 100%; height: 100%; object-fit: cover; opacity: 0.4; }
.overlay { position: relative; z-index: 2; }
.form-container { max-width: 700px; margin: 30px auto; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 5px 20px rgba(0,0,0,0.05); }
.form-group { margin-bottom: 30px; }
.form-group h3 { color: #ff6b6b; border-bottom: 1px solid #eee; padding-bottom: 10px; margin-bottom: 15px; }
.row { display: flex; gap: 15px; }
.row input, .row select { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 5px; }
.menu-item { border: 1px solid #eee; padding: 15px; border-radius: 5px; margin-bottom: 10px; cursor: pointer; }
.menu-item:hover { background: #f9f9f9; }
.menu-item.selected { border: 2px solid #ff6b6b; background: #fff5f5; }
.m-head { display: flex; justify-content: space-between; color: #333; margin-bottom: 5px; }
.extras-list { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.btn-reserve { width: 100%; background: #ff6b6b; color: white; border: none; padding: 15px; border-radius: 5px; font-size: 1.1rem; font-weight: bold; cursor: pointer; margin-top: 10px; }
.disclaimer { font-size: 0.8rem; color: #777; text-align: center; margin-top: 10px; }
.login-alert { text-align: center; margin-top: 50px; font-size: 1.2rem; color: #e74c3c; }

/* ADMIN */
.admin-view { padding: 40px; }
table { width: 100%; border-collapse: collapse; background: white; }
th, td { padding: 12px; border-bottom: 1px solid #ddd; text-align: left; }
th { background: #34495e; color: white; }
</style>