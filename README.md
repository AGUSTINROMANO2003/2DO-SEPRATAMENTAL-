# 2DO-SEPRATAMENTAL-


# ... (las de firebase y provider que ya teníamos) ...


# Para manejar datos de ubicación (aunque aún no usemos el mapa) geolocator: ^12.0.0
Ejecuta flutter pub get.
Paso 2: El "Modelo" de la Partera (La Tarjeta de Presentación) Necesitamos un "molde" para los datos de la partera.
Crea el archivo lib/models/midwife_profile.dart: Dardo
 
import 'package:cloud_firestore/cloud_firestore.dart';


// Este es el "molde" o la "tarjeta de presentación" de cada partera.
// Define qué datos esperamos recibir de Firestore. class MidwifeProfile {
final String id; final String alias;
final List<String> specialities; final List<String> services;
final String locationGeohash; // Por ahora solo lo leemos


MidwifeProfile({ required this.id,
required this.alias,
required this.specialities, required this.services,
required this.locationGeohash,
});


// Un "constructor" especial que sabe cómo tomar un documento
// de Firestore (que es un mapa) y convertirlo en nuestro objeto MidwifeProfile. factory MidwifeProfile.fromFirestore(DocumentSnapshot doc) {
Map<String, dynamic> data = doc.data() as Map<String, dynamic>;


return MidwifeProfile( id: doc.id,
 
alias: data['alias'] ?? 'Partera (sin alias)',
specialities: List<String>.from(data['specialities'] ?? []), services: List<String>.from(data['services'] ?? []),
locationGeohash: data['locationGeohash'] ?? '',
);
}
}
Paso 3: El "Mensajero" (El Servicio de Parteras)
Necesitamos una clase que sepa cómo ir a la "plaza comunitaria" (Firestore) y traer la lista de parteras.
Crea el archivo lib/services/midwife_service.dart:
Dardo
import 'package:cloud_firestore/cloud_firestore.dart'; import 'package:parteria_viva/models/midwife_profile.dart';

class MidwifeService {
// Obtenemos la referencia a la colección de perfiles en Firestore final CollectionReference _profilesCollection =
FirebaseFirestore.instance.collection('midwife_profiles');


// Esta es la función principal.
// Es "Future" porque "toma tiempo" ir a la nube y regresar. Future<List<MidwifeProfile>> getMidwives() async {
try {
// 1. Pedimos a Firestore los perfiles donde "isPublic" sea true QuerySnapshot snapshot = await _profilesCollection
 
.where('isPublic', isEqualTo: true)
.get();


// 2. "Mapeamos" (convertimos) cada documento en un objeto MidwifeProfile List<MidwifeProfile> midwives = snapshot.docs
.map((doc) => MidwifeProfile.fromFirestore(doc))
.toList();


return midwives;


} catch (e) {
// Si algo sale mal (sin internet, permisos, etc.), imprimimos el error
// y devolvemos una lista vacía. print("Error al obtener parteras: $e"); return [];
}
}


// (Aquí podríamos añadir 'getMidwifeDetails(id)' en el futuro)
}
Paso 4: La "Página del Directorio" (La Lista de Parteras) Aquí es donde la usuaria ve la lista.
Crea el archivo lib/pages/find_midwife_page.dart:
Dardo
import 'package:flutter/material.dart';
import 'package:parteria_viva/models/midwife_profile.dart';
 
import 'package:parteria_viva/services/midwife_service.dart';
import 'package:parteria_viva/pages/midwife_profile_page.dart'; // La crearemos después


class FindMidwifePage extends StatefulWidget { const FindMidwifePage({super.key});

@override
_FindMidwifePageState createState() => _FindMidwifePageState();
}


class _FindMidwifePageState extends State<FindMidwifePage> {
// Usamos un 'Future' para guardar el resultado de la llamada al servicio late Future<List<MidwifeProfile>> _midwivesFuture;
final MidwifeService _midwifeService = MidwifeService();


@override
void initState() { super.initState();
// En cuanto la página se "despierta", mandamos a buscar a las parteras
_midwivesFuture = _midwifeService.getMidwives();
}


@override
Widget build(BuildContext context) { return Scaffold(
appBar: AppBar(
 
title: const Text('Encuentra tu Partera'),
// (Aquí podríamos añadir el botón para cambiar a "Mapa")
),
// FutureBuilder es un widget mágico: muestra un "cargando" mientras
// espera que el "Future" (nuestro mensajero) regrese con los datos. body: FutureBuilder<List<MidwifeProfile>>(
future: _midwivesFuture,
builder: (context, snapshot) {
// --- Caso 1: Estamos esperando (cargando) ---
if (snapshot.connectionState == ConnectionState.waiting) { return const Center(child: CircularProgressIndicator());
}


// --- Caso 2: Hubo un error --- if (snapshot.hasError) {
return Center(child: Text('Error al cargar datos: ${snapshot.error}'));
}


// --- Caso 3: Tenemos datos, pero la lista está vacía --- if (!snapshot.hasData || snapshot.data!.isEmpty) {
return const Center(child: Text('No hay parteras disponibles en este momento.'));
}


// --- Caso 4: ¡Éxito! Tenemos la lista ---
List<MidwifeProfile> midwives = snapshot.data!;
 
// Usamos ListView.builder para crear la lista eficientemente return ListView.builder(
itemCount: midwives.length, itemBuilder: (context, index) { final midwife = midwives[index]; return Card(
margin: const EdgeInsets.symmetric(horizontal: 10, vertical: 5), child: ListTile(
title: Text(midwife.alias),
subtitle: Text(midwife.specialities.join(', ')), // Unimos las especialidades
leading: const Icon(Icons.person_pin_circle_outlined, size: 40, color: Colors.teal), trailing: const Icon(Icons.chevron_right),
onTap: () {
// Acción: Navegar a la página de detalle de esta partera Navigator.push(
context, MaterialPageRoute(
builder: (context) => MidwifeProfilePage(midwife: midwife),
),
);
},
),
);
},
);
},
 
),
);
}
}
Paso 5: La "Página de Perfil" (El Detalle)
Cuando la usuaria toca una partera, ve esto.
Crea el archivo lib/pages/midwife_profile_page.dart:
Dardo
import 'package:flutter/material.dart';
import 'package:parteria_viva/models/midwife_profile.dart';


class MidwifeProfilePage extends StatelessWidget {
// Esta página "recibe" el perfil de la partera que seleccionamos final MidwifeProfile midwife;

const MidwifeProfilePage({super.key, required this.midwife});


@override
Widget build(BuildContext context) { return Scaffold(
appBar: AppBar(
title: Text(midwife.alias),
),
body: Padding(
padding: const EdgeInsets.all(16.0),
child: ListView( // Usamos ListView para que se pueda hacer scroll
 
children: [
// (Aquí iría la foto de la partera) Center(
child: CircleAvatar( radius: 60,
backgroundColor: Colors.teal.shade100,
child: const Icon(Icons.person, size: 80, color: Colors.teal),
),
),
const SizedBox(height: 20), Text(
midwife.alias,
style: Theme.of(context).textTheme.headlineMedium, textAlign: TextAlign.center,
),
const SizedBox(height: 20),


_buildSectionTitle(context, 'Especialidades'),
Text(midwife.specialities.isEmpty ? 'No especificadas' : midwife.specialities.join(', ')), const Divider(height: 30),

_buildSectionTitle(context, 'Servicios'),
Text(midwife.services.isEmpty ? 'No especificados' : midwife.services.join(', ')), const Divider(height: 30),

_buildSectionTitle(context, 'Región de Atención'),
 
Text('Zona: ${midwife.locationGeohash} (Aprox.)'), // Mostramos la zona, no la ubicación exacta


const SizedBox(height: 40),


// Este es el botón MÁS IMPORTANTE de esta pantalla ElevatedButton.icon(
icon: const Icon(Icons.calendar_today), label: const Text('Solicitar Cita'),
style: ElevatedButton.styleFrom(
padding: const EdgeInsets.symmetric(vertical: 15), backgroundColor: Colors.teal,
foregroundColor: Colors.white,
textStyle: const TextStyle(fontSize: 18)
),
onPressed: () {
// TODO: Navegar a la pantalla de "Agendar Cita" ScaffoldMessenger.of(context).showSnackBar(
const SnackBar(content: Text('Próximamente: Agendar cita'))
);
},
)
],
),
),
);
 
}


// Un pequeño widget de ayuda para los títulos de sección Widget _buildSectionTitle(BuildContext context, String title) { return Padding(
padding: const EdgeInsets.only(bottom: 8.0), child: Text(
title.toUpperCase(),
style: Theme.of(context).textTheme.titleSmall?.copyWith( color: Colors.teal.shade700,
fontWeight: FontWeight.bold, letterSpacing: 0.8
),
),
);
}
}
Paso 6: Conecta el Módulo (Actualiza la Página de Inicio)
Finalmente, necesitamos un botón en nuestra home_page.dart(que creamos en el Módulo 1) para ir a esta nueva sección.
Abre lib/pages/home_page.darty modifica:
Dardo
import 'package:flutter/material.dart'; import 'package:provider/provider.dart';
import 'package:parteria_viva/services/auth_service.dart';
import 'package:parteria_viva/pages/find_midwife_page.dart'; // <-- ¡Importa la nueva página!
 
class HomePage extends StatelessWidget { const HomePage({super.key});

@override
Widget build(BuildContext context) {
final authService = Provider.of<AuthService>(context);


return Scaffold( appBar: AppBar(
title: const Text('Mi Plan de Cuidados'), actions: [
IconButton(
icon: const Icon(Icons.logout), onPressed: () {
authService.signOut();
},
),
],
),
body: Center( child: Padding(
padding: const EdgeInsets.all(20.0), child: Column(
mainAxisAlignment: MainAxisAlignment.center, children: [
 
Text(
'¡Bienvenida a Partería Viva!',
style: Theme.of(context).textTheme.headlineSmall, textAlign: TextAlign.center,
),
const SizedBox(height: 40),


// --- NUEVO BOTÓN ---
ElevatedButton.icon(
icon: const Icon(Icons.search),
label: const Text('Buscar una Partera'), style: ElevatedButton.styleFrom(
minimumSize: const Size(double.infinity, 50),
),
onPressed: () { Navigator.push( context,
MaterialPageRoute(builder: (context) => const FindMidwifePage()),
);
},
),
// --- FIN NUEVO BOTÓN ---


const SizedBox(height: 20),


// (Aquí irán más botones: "Mi Plan", "Mi Bitácora", etc.)
 
],
),
),
),
);
}
}
