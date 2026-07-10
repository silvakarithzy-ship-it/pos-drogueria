const express = require('express');
const mysql = require('mysql2');
const cors = require('cors');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const socketIo = require('socket.io');
const http = require('http');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());

// Conexión a MySQL
const db = mysql.createConnection({
  host: 'localhost',
  user: 'pos_user',
  password: 'tu_password',
  database: 'pos_drogueria'
});

// Login de usuarios
app.post('/login', (req, res) => {
  const { email, password } = req.body;
  db.query('SELECT * FROM usuarios WHERE email = ?', [email], async (err, result) => {
    if (err) return res.status(500).send(err);
    if (result.length === 0) return res.status(401).send('Usuario no encontrado');
    const match = await bcrypt.compare(password, result[0].password);
    if (!match) return res.status(401).send('Contraseña incorrecta');
    const token = jwt.sign({ id: result[0].id, role: result[0].role }, 'secretkey');
    res.json({ token });
  });
});

// Inventario
app.get('/productos', (req, res) => {
  db.query('SELECT * FROM productos', (err, result) => {
    if (err) return res.status(500).send(err);
    res.json(result);
  });
});

// Ventas
app.post('/ventas', (req, res) => {
  const { productos, total } = req.body;
  db.query('INSERT INTO ventas (total) VALUES (?)', [total], (err, result) => {
    if (err) return res.status(500).send(err);
    const ventaId = result.insertId;
    productos.forEach(p => {
      db.query('INSERT INTO detalle_ventas (venta_id, producto_id, cantidad) VALUES (?, ?, ?)',
        [ventaId, p.id, p.cantidad]);
    });
    io.emit('nuevaVenta', { ventaId, total });
    res.json({ message: 'Venta registrada', ventaId });
  });
});

// Caja
app.post('/caja', (req, res) => {
  const { tipo, monto, descripcion } = req.body;
  db.query('INSERT INTO caja (tipo, monto, descripcion) VALUES (?, ?, ?)', [tipo, monto, descripcion], (err) => {
    if (err) return res.status(500).send(err);
    res.json({ message: 'Movimiento registrado' });
  });
});

// Notificaciones
app.post('/notificaciones', (req, res) => {
  const { mensaje, tipo } = req.body;
  db.query('INSERT INTO historial_notificaciones (mensaje, tipo) VALUES (?, ?)', [mensaje, tipo], (err) => {
    if (err) return res.status(500).send(err);
    io.emit('nuevaNotificacion', { mensaje, tipo });
    res.json({ message: 'Notificación enviada' });
  });
});

server.listen(3000, () => {
  console.log('Servidor corriendo en puerto 3000');
});

