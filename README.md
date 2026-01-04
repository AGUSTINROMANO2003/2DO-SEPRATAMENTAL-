# 2DO-SEPRATAMENTAL-


import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:provider/provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const MyApp());
}

/// ─────────────────────────────────────────────
/// APP PRINCIPAL
/// ─────────────────────────────────────────────
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        Provider<AuthService>(
          create: (_) => AuthService(),
        ),
      ],
      child: MaterialApp(
        debugShowCheckedModeBanner: false,
        title: 'Partería Viva',
        theme: ThemeData(
          primarySwatch: Colors.teal,
        ),
        home: const HomePage(),
      ),
    );
  }
}

/// ─────────────────────────────────────────────
/// AUTH SERVICE (BÁSICO)
/// ─────────────────────────────────────────────
class AuthService {
  void signOut() {
    print('Sesión cerrada');
  }
}

/// ─────────────────────────────────────────────
/// MODELO MIDWIFE
/// ─────────────────────────────────────────────
class MidwifeProfile {
  final String id;
  final String alias;
  final List<String> specialities;
  final List<String> services;
  final String locationGeohash;

  MidwifeProfile({
    required this.id,
    required this.alias,
    required this.specialities,
    required this.services,
    required this.locationGeohash,
  });

  factory MidwifeProfile.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;

    return MidwifeProfile(
      id: doc.id,
      alias: data['alias'] ?? 'Partera sin alias',
      specialities: List<String>.from(data['specialities'] ?? []),
      services: List<String>.from(data['services'] ?? []),
      locationGeohash: data['locationGeohash'] ?? '',
    );
  }
}

/// ─────────────────────────────────────────────
/// SERVICIO FIRESTORE
/// ─────────────────────────────────────────────
class MidwifeService {
  final CollectionReference _collection =
      FirebaseFirestore.instance.collection('midwife_profiles');

  Future<List<MidwifeProfile>> getMidwives() async {
    try {
      final snapshot =
          await _collection.where('isPublic', isEqualTo: true).get();

      return snapshot.docs
          .map((doc) => MidwifeProfile.fromFirestore(doc))
          .toList();
    } catch (e) {
      print('Error al obtener parteras: $e');
      return [];
    }
  }
}

/// ─────────────────────────────────────────────
/// HOME PAGE
/// ─────────────────────────────────────────────
class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    final authService = Provider.of<AuthService>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Mi Plan de Cuidados'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () => authService.signOut(),
          ),
        ],
      ),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(20),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text(
                '¡Bienvenida a Partería Viva!',
                style: Theme.of(context).textTheme.headlineSmall,
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 40),
              ElevatedButton.icon(
                icon: const Icon(Icons.search),
                label: const Text('Buscar una Partera'),
                style: ElevatedButton.styleFrom(
                  minimumSize: const Size(double.infinity, 50),
                ),
                onPressed: () {
                  Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (_) => const FindMidwifePage(),
                    ),
                  );
                },
              ),
            ],
          ),
        ),
      ),
    );
  }
}

/// ─────────────────────────────────────────────
/// LISTA DE PARTERAS
/// ─────────────────────────────────────────────
class FindMidwifePage extends StatefulWidget {
  const FindMidwifePage({super.key});

  @override
  State<FindMidwifePage> createState() => _FindMidwifePageState();
}

class _FindMidwifePageState extends State<FindMidwifePage> {
  final MidwifeService _service = MidwifeService();
  late Future<List<MidwifeProfile>> _future;

  @override
  void initState() {
    super.initState();
    _future = _service.getMidwives();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Encuentra tu Partera'),
      ),
      body: FutureBuilder<List<MidwifeProfile>>(
        future: _future,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }

          if (!snapshot.hasData || snapshot.data!.isEmpty) {
            return const Center(
              child: Text('No hay parteras disponibles'),
            );
          }

          final midwives = snapshot.data!;

          return ListView.builder(
            itemCount: midwives.length,
            itemBuilder: (context, index) {
              final midwife = midwives[index];
              return Card(
                margin:
                    const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
                child: ListTile(
                  leading: const Icon(
                    Icons.person_pin_circle_outlined,
                    color: Colors.teal,
                    size: 40,
                  ),
                  title: Text(midwife.alias),
                  subtitle: Text(midwife.specialities.join(', ')),
                  trailing: const Icon(Icons.chevron_right),
                  onTap: () {
                    Navigator.push(
                      context,
                      MaterialPageRoute(
                        builder: (_) =>
                            MidwifeProfilePage(midwife: midwife),
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

/// ─────────────────────────────────────────────
/// PERFIL DE PARTERA
/// ─────────────────────────────────────────────
class MidwifeProfilePage extends StatelessWidget {
  final MidwifeProfile midwife;

  const MidwifeProfilePage({super.key, required this.midwife});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(midwife.alias),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: ListView(
          children: [
            Center(
              child: CircleAvatar(
                radius: 60,
                backgroundColor: Colors.teal.shade100,
                child: const Icon(
                  Icons.person,
                  size: 80,
                  color: Colors.teal,
                ),
              ),
            ),
            const SizedBox(height: 20),
            Text(
              midwife.alias,
              textAlign: TextAlign.center,
              style: Theme.of(context).textTheme.headlineMedium,
            ),
            const SizedBox(height: 20),
            _sectionTitle(context, 'Especialidades'),
            Text(midwife.specialities.join(', ')),
            const Divider(height: 30),
            _sectionTitle(context, 'Servicios'),
            Text(midwife.services.join(', ')),
            const Divider(height: 30),
            _sectionTitle(context, 'Región de Atención'),
            Text('Zona aproximada: ${midwife.locationGeohash}'),
            const SizedBox(height: 40),
            ElevatedButton.icon(
              icon: const Icon(Icons.calendar_today),
              label: const Text('Solicitar Cita'),
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.teal,
                foregroundColor: Colors.white,
                padding: const EdgeInsets.symmetric(vertical: 15),
                textStyle: const TextStyle(fontSize: 18),
              ),
              onPressed: () {
                ScaffoldMessenger.of(context).showSnackBar(
                  const SnackBar(
                    content: Text('Próximamente: Agendar cita'),
                  ),
                );
              },
            ),
          ],
        ),
      ),
    );
  }

  Widget _sectionTitle(BuildContext context, String title) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 8),
      child: Text(
        title.toUpperCase(),
        style: Theme.of(context).textTheme.titleSmall?.copyWith(
              color: Colors.teal.shade700,
              fontWeight: FontWeight.bold,
              letterSpacing: 0.8,
            ),
      ),
    );
  }
}


