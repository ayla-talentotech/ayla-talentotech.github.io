flutter create planet_manager
cd planet_manager
dependencies:
  flutter:
    sdk: flutter
  sqflite: ^2.0.0+3
  path: ^1.8.0
class Planet {
  int? id;
  String name;
  double distanceFromSun;
  double size;
  String? nickname;

  Planet({this.id, required this.name, required this.distanceFromSun, required this.size, this.nickname});

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'name': name,
      'distanceFromSun': distanceFromSun,
      'size': size,
      'nickname': nickname,
    };
  }

  factory Planet.fromMap(Map<String, dynamic> map) {
    return Planet(
      id: map['id'],
      name: map['name'],
      distanceFromSun: map['distanceFromSun'],
      size: map['size'],
      nickname: map['nickname'],
    );
  }
}
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();

  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;

    _database = await _initDB('planets.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future _createDB(Database db, int version) async {
    const idType = 'INTEGER PRIMARY KEY AUTOINCREMENT';
    const textType = 'TEXT NOT NULL';
    const doubleType = 'REAL NOT NULL';

    await db.execute('''
CREATE TABLE planets (
  id $idType,
  name $textType,
  distanceFromSun $doubleType,
  size $doubleType,
  nickname TEXT
)
''');
  }

  Future<Planet> create(Planet planet) async {
    final db = await instance.database;

    final id = await db.insert('planets', planet.toMap());
    return planet.copyWith(id: id);
  }

  Future<Planet> readPlanet(int id) async {
    final db = await instance.database;

    final maps = await db.query(
      'planets',
      columns: ['id', 'name', 'distanceFromSun', 'size', 'nickname'],
      where: 'id = ?',
      whereArgs: [id],
    );

    if (maps.isNotEmpty) {
      return Planet.fromMap(maps.first);
    } else {
      throw Exception('ID $id not found');
    }
  }

  Future<List<Planet>> readAllPlanets() async {
    final db = await instance.database;

    const orderBy = 'name ASC';
    final result = await db.query('planets', orderBy: orderBy);

    return result.map((json) => Planet.fromMap(json)).toList();
  }

  Future<int> update(Planet planet) async {
    final db = await instance.database;

    return db.update(
      'planets',
      planet.toMap(),
      where: 'id = ?',
      whereArgs: [planet.id],
    );
  }

  Future<int> delete(int id) async {
    final db = await instance.database;

    return await db.delete(
      'planets',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}
import 'package:flutter/material.dart';
import 'package:your_app/database_helper.dart';
import 'package:your_app/planet.dart';
import 'package:your_app/add_edit_planet_page.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gerenciador de Planetas',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: PlanetListPage(),
    );
  }
}

class PlanetListPage extends StatefulWidget {
  @override
  _PlanetListPageState createState() => _PlanetListPageState();
}

class _PlanetListPageState extends State<PlanetListPage> {
  late List<Planet> planets;

  @override
  void initState() {
    super.initState();
    refreshPlanetList();
  }

  Future refreshPlanetList() async {
    planets = await DatabaseHelper.instance.readAllPlanets();
    setState(() {});
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Planetas'),
      ),
      body: ListView.builder(
        itemCount: planets.length,
        itemBuilder: (context, index) {
          final planet = planets[index];
          return ListTile(
            title: Text(planet.name),
            subtitle: Text(planet.nickname ?? ''),
            onTap: () => Navigator.of(context).push(MaterialPageRoute(
              builder: (context) => AddEditPlanetPage(planet: planet),
            )),
            trailing: IconButton(
              icon: Icon(Icons.delete),
              onPressed: () async {
                await DatabaseHelper.instance.delete(planet.id!);
                refreshPlanetList();
              },
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => Navigator.of(context).push(MaterialPageRoute(
          builder: (context) => AddEditPlanetPage(),
        )),
      ),
    );
  }
}
