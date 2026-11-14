import React, { useState, useEffect } from 'react';
import { 
  StyleSheet, Text, View, FlatList, Image, 
  TouchableOpacity, Modal, TextInput, Alert, ScrollView 
} from 'react-native';
import { createClient } from '@supabase/supabase-js';

// Configura las credenciales de tu proyecto en Supabase
const SUPABASE_URL = 'https://TU_PROYECTO.supabase.co';
const SUPABASE_ANON_KEY = 'TU_ANON_KEY';
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

export default function App() {
  // Estados de la app
  const [usuario, setUsuario] = useState({
    id: '123e4567-e89b-12d3-a456-426614174000', // Ejemplo UUID
    nombre_completo: 'Brandon González',
    edad: 21,
    tarjeta_credito: '**** **** **** 4589', // Máscara de privacidad
    fotografia_url: 'https://via.placeholder.com/150',
    rol: 'admin' // 'admin' o 'cliente'
  });

  const [peliculas, setPeliculas] = useState([]);
  const [rentas, setRentas] = useState([]);
  const [modalPerfil, setModalPerfil] = useState(false);
  const [modalBitacora, setModalBitacora] = useState(false);
  const [modalNuevaPeli, setModalNuevaPeli] = useState(false);

  // Formulario de nueva película (Admin)
  const [nuevoTitulo, setNuevoTitulo] = useState('');
  const [nuevoGenero, setNuevoGenero] = useState('');
  const [nuevaImagen, setNuevaImagen] = useState('');

  useEffect(() => {
    cargarPeliculas();
    cargarHistorialRentas();
  }, []);

  const cargarPeliculas = async () => {
    const { data, error } = await supabase.from('catalogo_peliculas').select('*');
    if (!error && data) setPeliculas(data);
  };

  const cargarHistorialRentas = async () => {
    const { data, error } = await supabase
      .from('historial_rentas')
      .select('*, perfiles_usuario(nombre_completo), catalogo_peliculas(titulo)');
    if (!error && data) setRentas(data);
  };

  // Función para procesar una renta
  const procesarRenta = async (peliculaId) => {
    const fechaInicio = new Date();
    const fechaFin = new Date();
    fechaFin.setDate(fechaInicio.getDate() + 5); // 5 días de renta

    const { error } = await supabase.from('historial_rentas').insert([{
      usuario_id: usuario.id,
      pelicula_id: peliculaId,
      fecha_inicio: fechaInicio.toISOString(),
      fecha_fin: fechaFin.toISOString()
    }]);

    if (error) {
      Alert.alert('Error', 'No se pudo procesar la renta');
    } else {
      Alert.alert('¡Éxito!', 'La película ha sido rentada correctamente.');
      cargarHistorialRentas();
    }
  };

  // Funciones de Administrador
  const agregarPelicula = async () => {
    if (!nuevoTitulo || !nuevoGenero) return;
    const { error } = await supabase.from('catalogo_peliculas').insert([{
      titulo: nuevoTitulo,
      genero: nuevoGenero,
      poster_url: nuevaImagen || 'https://via.placeholder.com/300x450'
    }]);

    if (!error) {
      setModalNuevaPeli(false);
      setNuevoTitulo('');
      setNuevoGenero('');
      setNuevaImagen('');
      cargarPeliculas();
    }
  };

  const eliminarPelicula = async (id) => {
    const { error } = await supabase.from('catalogo_peliculas').delete().eq('id', id);
    if (!error) cargarPeliculas();
  };

  return (
    <View style={styles.container}>
      {/* Encabezado Superior */}
      <View style={styles.header}>
        <TouchableOpacity onPress={() => setModalPerfil(true)} style={styles.profileBadge}>
          <Image source={{ uri: usuario.fotografia_url }} style={styles.avatarHeader} />
          <Text style={styles.headerTitle}>{usuario.nombre_completo}</Text>
        </TouchableOpacity>

        {usuario.rol === 'admin' && (
          <TouchableOpacity 
            style={styles.adminBtn} 
            onPress={() => setModalBitacora(true)}>
            <Text style={styles.adminBtnText}>📋 Bitácora Admin</Text>
          </TouchableOpacity>
        )}
      </View>

      <Text style={styles.sectionTitle}>Cartelera Disponible</Text>

      {/* Botón para agregar película si es Admin[cite: 1] */}
      {usuario.rol === 'admin' && (
        <TouchableOpacity style={styles.addBtn} onPress={() => setModalNuevaPeli(true)}>
          <Text style={styles.addBtnText}>+ Agregar Nueva Película</Text>
        </TouchableOpacity>
      )}

      {/* Lista del Catálogo */}
      <FlatList
        data={peliculas}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => {
          const rentaActiva = rentas.find(r => r.pelicula_id === item.id && r.usuario_id === usuario.id);

          return (
            <View style={styles.card}>
              <Image source={{ uri: item.poster_url }} style={styles.poster} />
              <View style={styles.cardContent}>
                <Text style={styles.movieTitle}>{item.titulo}</Text>
                <Text style={styles.movieGenre}>{item.genero}</Text>

                {rentaActiva ? (
                  <View style={styles.rentedBadge}>
                    <Text style={styles.rentedText}>✓ En tu Posesión</Text>
                    <Text style={styles.rentedSubtext}>
                      Vence: {new Date(rentaActiva.fecha_fin).toLocaleDateString()}
                    </Text>
                  </View>
                ) : (
                  <TouchableOpacity 
                    style={styles.rentBtn} 
                    onPress={() => procesarRenta(item.id)}>
                    <Text style={styles.rentBtnText}>Rentar Película</Text>
                  </TouchableOpacity>
                )}

                {usuario.rol === 'admin' && (
                  <TouchableOpacity 
                    style={styles.deleteBtn} 
                    onPress={() => eliminarPelicula(item.id)}>
                    <Text style={styles.deleteBtnText}>Eliminar de Cartelera</Text>
                  </TouchableOpacity>
                )}
              </View>
            </View>
          );
        }}
      />

      {/* Modal 1: Perfil de Usuario[cite: 1] */}
      <Modal visible={modalPerfil} animationType="slide">
        <View style={styles.modalContainer}>
          <Text style={styles.modalTitle}>Perfil de Usuario</Text>
          <Image source={{ uri: usuario.fotografia_url }} style={styles.avatarLarge} />
          
          <View style={styles.infoBox}>
            <Text style={styles.infoLabel}>Nombre Completo:</Text>
            <Text style={styles.infoValue}>{usuario.nombre_completo}</Text>
            
            <Text style={styles.infoLabel}>Edad:</Text>
            <Text style={styles.infoValue}>{usuario.edad} años</Text>
            
            <Text style={styles.infoLabel}>Tarjeta registrada:</Text>
            <Text style={styles.infoValue}>{usuario.tarjeta_credito}</Text>
            
            <Text style={styles.infoLabel}>Rol del Sistema:</Text>
            <Text style={styles.infoValue}>{usuario.rol.toUpperCase()}</Text>
          </View>

          <TouchableOpacity style={styles.closeBtn} onPress={() => setModalPerfil(false)}>
            <Text style={styles.closeBtnText}>Cerrar Perfil</Text>
          </TouchableOpacity>
        </View>
      </Modal>

      {/* Modal 2: Bitácora / Reporte de Rentas Admin[cite: 1] */}
      <Modal visible={modalBitacora} animationType="fade">
        <View style={styles.modalContainer}>
          <Text style={styles.modalTitle}>Informe de Rentas (Admin)</Text>
          <ScrollView style={{ width: '100%' }}>
            {rentas.map((r, i) => (
              <View key={i} style={styles.logCard}>
                <Text style={styles.logTitle}>{r.catalogo_peliculas?.titulo}</Text>
                <Text style={styles.logUser}>Cliente: {r.perfiles_usuario?.nombre_completo}</Text>
                <Text style={styles.logDate}>Inicio: {new Date(r.fecha_inicio).toLocaleDateString()}</Text>
                <Text style={styles.logDate}>Fin: {new Date(r.fecha_fin).toLocaleDateString()}</Text>
              </View>
            ))}
          </ScrollView>
          <TouchableOpacity style={styles.closeBtn} onPress={() => setModalBitacora(false)}>
            <Text style={styles.closeBtnText}>Volver</Text>
          </TouchableOpacity>
        </View>
      </Modal>

      {/* Modal 3: Agregar Película Admin[cite: 1] */}
      <Modal visible={modalNuevaPeli} animationType="slide">
        <View style={styles.modalContainer}>
          <Text style={styles.modalTitle}>Agregar Película</Text>
          <TextInput 
            placeholder="Título" 
            style={styles.input} 
            value={nuevoTitulo} 
            onChangeText={setNuevoTitulo} 
          />
          <TextInput 
            placeholder="Género" 
            style={styles.input} 
            value={nuevoGenero} 
            onChangeText={setNuevoGenero} 
          />
          <TextInput 
            placeholder="URL Imagen (Póster)" 
            style={styles.input} 
            value={nuevaImagen} 
            onChangeText={setNuevaImagen} 
          />
          <TouchableOpacity style={styles.addBtn} onPress={agregarPelicula}>
            <Text style={styles.addBtnText}>Guardar Película</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.closeBtn} onPress={() => setModalNuevaPeli(false)}>
            <Text style={styles.closeBtnText}>Cancelar</Text>
          </TouchableOpacity>
        </View>
      </Modal>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#0f172a', paddingTop: 50, paddingHorizontal: 16 },
  header: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 20 },
  profileBadge: { flexDirection: 'row', alignItems: 'center' },
  avatarHeader: { width: 40, height: 40, borderRadius: 20, marginRight: 10 },
  headerTitle: { color: '#ffffff', fontSize: 16, fontWeight: 'bold' },
  adminBtn: { backgroundColor: '#3b82f6', padding: 8, borderRadius: 8 },
  adminBtnText: { color: '#ffffff', fontWeight: '600' },
  sectionTitle: { color: '#f8fafc', fontSize: 22, fontWeight: 'bold', marginBottom: 15 },
  addBtn: { backgroundColor: '#10b981', padding: 12, borderRadius: 8, marginBottom: 15, alignItems: 'center' },
  addBtnText: { color: '#ffffff', fontWeight: 'bold' },
  card: { flexDirection: 'row', backgroundColor: '#1e293b', borderRadius: 12, marginBottom: 15, padding: 10 },
  poster: { width: 90, height: 130, borderRadius: 8 },
  cardContent: { flex: 1, marginLeft: 15, justifyContent: 'space-around' },
  movieTitle: { color: '#ffffff', fontSize: 18, fontWeight: 'bold' },
  movieGenre: { color: '#94a3b8', fontSize: 14 },
  rentBtn: { backgroundColor: '#6366f1', paddingVertical: 8, borderRadius: 6, alignItems: 'center' },
  rentBtnText: { color: '#ffffff', fontWeight: 'bold' },
  rentedBadge: { backgroundColor: '#065f46', padding: 6, borderRadius: 6 },
  rentedText: { color: '#34d399', fontWeight: 'bold', fontSize: 12 },
  rentedSubtext: { color: '#a7f3d0', fontSize: 10 },
  deleteBtn: { marginTop: 6, alignItems: 'center' },
  deleteBtnText: { color: '#ef4444', fontSize: 12 },
  modalContainer: { flex: 1, backgroundColor: '#0f172a', padding: 25, justifyContent: 'center', alignItems: 'center' },
  modalTitle: { color: '#ffffff', fontSize: 24, fontWeight: 'bold', marginBottom: 20 },
  avatarLarge: { width: 100, height: 100, borderRadius: 50, marginBottom: 20 },
  infoBox: { width: '100%', backgroundColor: '#1e293b', padding: 20, borderRadius: 12, marginBottom: 20 },
  infoLabel: { color: '#94a3b8', fontSize: 12, marginTop: 8 },
  infoValue: { color: '#ffffff', fontSize: 16, fontWeight: '600' },
  closeBtn: { backgroundColor: '#475569', padding: 12, borderRadius: 8, width: '100%', alignItems: 'center', marginTop: 10 },
  closeBtnText: { color: '#ffffff', fontWeight: 'bold' },
  input: { backgroundColor: '#1e293b', color: '#ffffff', width: '100%', padding: 12, borderRadius: 8, marginBottom: 12 },
  logCard: { backgroundColor: '#1e293b', padding: 15, borderRadius: 8, marginBottom: 10, width: '100%' },
  logTitle: { color: '#38bdf8', fontSize: 16, fontWeight: 'bold' },
  logUser: { color: '#ffffff', fontSize: 14 },
  logDate: { color: '#94a3b8', fontSize: 12 }
});
