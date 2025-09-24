# Fitness-tracking-app
spezial fitness tracking app
# Flutter Bodybuilding App - Starter Project

Dieses Dokument enthält ein lauffähiges Starter-Template für eine mobile Bodybuilding-Trainings-App (Flutter).

### Projektstruktur (einfügbar in ein neues Flutter-Projekt)

```
pubspec.yaml
lib/
  main.dart
  models/
    exercise.dart
    workout_entry.dart
  db/
    database.dart
  providers/
    workout_provider.dart
  screens/
    home_screen.dart
    workout_screen.dart
    history_screen.dart
  widgets/
    exercise_row.dart

README.md
```

---

## pubspec.yaml

```yaml
name: bodybuilding_app
description: Starter App zum Tracken von Bodybuilding-Workouts (Gewicht, Wiederholungen, Splits)
publish_to: 'none'
version: 0.1.0
environment:
  sdk: '>=2.18.0 <3.0.0'

dependencies:
  flutter:
    sdk: flutter
  provider: ^6.0.5
  sqflite: ^2.2.8
  path_provider: ^2.0.14
  path: ^1.8.2
  intl: ^0.18.0

flutter:
  uses-material-design: true
```

---

## lib/main.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'providers/workout_provider.dart';
import 'screens/home_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => WorkoutProvider()..init(),
      child: MaterialApp(
        title: 'Bodybuilding Tracker',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: const HomeScreen(),
      ),
    );
  }
}
```

---

## lib/models/exercise.dart

```dart
class Exercise {
  final int? id;
  final String name;
  final String muscleGroup; // z.B. Brust, Schulter, Lat, Beine, Bizeps, Trizeps

  Exercise({this.id, required this.name, required this.muscleGroup});

  Map<String, dynamic> toMap() => {
        'id': id,
        'name': name,
        'muscleGroup': muscleGroup,
      };

  factory Exercise.fromMap(Map<String, dynamic> m) => Exercise(
        id: m['id'] as int?,
        name: m['name'] as String,
        muscleGroup: m['muscleGroup'] as String,
      );
}
```

---

## lib/models/workout_entry.dart

```dart
class WorkoutEntry {
  final int? id;
  final String date; // ISO string yyyy-MM-dd
  final String exercise; // name
  final int sets;
  final int reps;
  final double weight; // kg
  final String note;

  WorkoutEntry({this.id, required this.date, required this.exercise, required this.sets, required this.reps, required this.weight, this.note = ''});

  Map<String, dynamic> toMap() => {
        'id': id,
        'date': date,
        'exercise': exercise,
        'sets': sets,
        'reps': reps,
        'weight': weight,
        'note': note,
      };

  factory WorkoutEntry.fromMap(Map<String, dynamic> m) => WorkoutEntry(
        id: m['id'] as int?,
        date: m['date'] as String,
        exercise: m['exercise'] as String,
        sets: m['sets'] as int,
        reps: m['reps'] as int,
        weight: (m['weight'] as num).toDouble(),
        note: m['note'] as String? ?? '',
      );
}
```

---

## lib/db/database.dart

```dart
import 'dart:async';
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';
import '../models/exercise.dart';
import '../models/workout_entry.dart';

class AppDatabase {
  static final AppDatabase _instance = AppDatabase._internal();
  factory AppDatabase() => _instance;
  AppDatabase._internal();

  static Database? _db;

  Future<Database> get db async {
    if (_db != null) return _db!;
    _db = await _init();
    return _db!;
  }

  Future<Database> _init() async {
    final databasesPath = await getDatabasesPath();
    final path = join(databasesPath, 'bodybuilding.db');

    return openDatabase(path, version: 1, onCreate: _onCreate);
  }

  FutureOr<void> _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE exercises(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        muscleGroup TEXT NOT NULL
      )
    ''');

    await db.execute('''
      CREATE TABLE entries(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        date TEXT NOT NULL,
        exercise TEXT NOT NULL,
        sets INTEGER NOT NULL,
        reps INTEGER NOT NULL,
        weight REAL NOT NULL,
        note TEXT
      )
    ''');

    // seed with common exercises
    final seed = [
      Exercise(name: 'Bankdrücken', muscleGroup: 'Brust'),
      Exercise(name: 'Schrägbank Kurzhantel', muscleGroup: 'Brust'),
      Exercise(name: 'Schulterdrücken', muscleGroup: 'Schulter'),
      Exercise(name: 'Seitheben', muscleGroup: 'Schulter'),
      Exercise(name: 'Klimmzüge', muscleGroup: 'Latissimus'),
      Exercise(name: 'Langhantelrudern', muscleGroup: 'Latissimus'),
      Exercise(name: 'Kniebeuge', muscleGroup: 'Beine'),
      Exercise(name: 'Beinpresse', muscleGroup: 'Beine'),
      Exercise(name: 'Bizepscurls', muscleGroup: 'Bizeps'),
      Exercise(name: 'Trizepsdrücken', muscleGroup: 'Trizeps'),
    ];

    for (final e in seed) {
      await db.insert('exercises', e.toMap());
    }
  }

  // Exercises
  Future<List<Exercise>> getExercises() async {
    final d = await db;
    final res = await d.query('exercises', orderBy: 'name');
    return res.map((m) => Exercise.fromMap(m)).toList();
  }

  Future<int> insertExercise(Exercise e) async {
    final d = await db;
    return d.insert('exercises', e.toMap());
  }

  // Entries
  Future<int> insertEntry(WorkoutEntry entry) async {
    final d = await db;
    return d.insert('entries', entry.toMap());
  }

  Future<List<WorkoutEntry>> entriesForExercise(String exercise) async {
    final d = await db;
    final res = await d.query('entries', where: 'exercise = ?', whereArgs: [exercise], orderBy: 'date DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }

  Future<List<WorkoutEntry>> entriesForDate(String date) async {
    final d = await db;
    final res = await d.query('entries', where: 'date = ?', whereArgs: [date], orderBy: 'id DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }

  Future<List<WorkoutEntry>> allEntries() async {
    final d = await db;
    final res = await d.query('entries', orderBy: 'date DESC, id DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }
}
```

---

## lib/providers/workout_provider.dart

```dart
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';
import '../db/database.dart';
import '../models/exercise.dart';
import '../models/workout_entry.dart';

class WorkoutProvider extends ChangeNotifier {
  final AppDatabase _db = AppDatabase();
  List<Exercise> exercises = [];
  List<WorkoutEntry> todaysEntries = [];
  List<WorkoutEntry> history = [];

  Future<void> init() async {
    exercises = await _db.getExercises();
    await loadToday();
    history = await _db.allEntries();
    notifyListeners();
  }

  String todayISO() => DateFormat('yyyy-MM-dd').format(DateTime.now());

  Future<void> loadToday() async {
    final date = todayISO();
    todaysEntries = await _db.entriesForDate(date);
    notifyListeners();
  }

  Future<void> addExercise(String name, String muscle) async {
    final e = Exercise(name: name, muscleGroup: muscle);
    await _db.insertExercise(e);
    exercises = await _db.getExercises();
    notifyListeners();
  }

  Future<void> addEntry(String exercise, int sets, int reps, double weight, {String note = ''}) async {
    final entry = WorkoutEntry(date: todayISO(), exercise: exercise, sets: sets, reps: reps, weight: weight, note: note);
    await _db.insertEntry(entry);
    await loadToday();
    history = await _db.allEntries();
    notifyListeners();
  }

  Future<List<WorkoutEntry>> lastEntriesForExercise(String exercise, {int limit = 3}) async {
    final list = await _db.entriesForExercise(exercise);
    return list.take(limit).toList();
  }
}
```

---

## lib/widgets/exercise_row.dart

```dart
import 'package:flutter/material.dart';
import '../models/exercise.dart';

class ExerciseRow extends StatelessWidget {
  final Exercise exercise;
  final VoidCallback? onTap;

  const ExerciseRow({super.key, required this.exercise, this.onTap});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(exercise.name),
      subtitle: Text(exercise.muscleGroup),
      trailing: const Icon(Icons.chevron_right),
      onTap: onTap,
    );
  }
}
```

---

## lib/screens/home_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/workout_provider.dart';
import 'workout_screen.dart';
import 'history_screen.dart';
import '../widgets/exercise_row.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);

    // Beispiel-Split (konfigurierbar in einer realen App)
    final split = {
      'Montag': ['Bankdrücken', 'Schulterdrücken', 'Trizepsdrücken'],
      'Dienstag': ['Klimmzüge', 'Bizepscurls', 'Kniebeuge'],
      'Mittwoch': ['Pause'],
      'Donnerstag': ['Bankdrücken', 'Rudern', 'Beinpresse'],
      'Freitag': ['Schulterdrücken', 'Seitheben', 'Trizepsdrücken'],
    };

    return Scaffold(
      appBar: AppBar(
        title: const Text('Bodybuilding Tracker'),
        actions: [
          IconButton(
            icon: const Icon(Icons.history),
            onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const HistoryScreen())),
          )
        ],
      ),
      body: RefreshIndicator(
        onRefresh: prov.init,
        child: ListView(
          padding: const EdgeInsets.all(12),
          children: [
            const Text('Trainings-Split (Beispiel)', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            ...split.entries.map((e) => Card(
                  child: ListTile(
                    title: Text(e.key),
                    subtitle: Text(e.value.join(', ')),
                    onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => WorkoutScreen(day: e.key, plannedExercises: e.value))),
                  ),
                )),

            const SizedBox(height: 16),
            const Text('Übungsbibliothek', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            ...prov.exercises.map((ex) => ExerciseRow(
                  exercise: ex,
                  onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => WorkoutScreen(day: 'Freies Training', plannedExercises: [ex.name]))),
                )),

            const SizedBox(height: 12),
            ElevatedButton.icon(
              icon: const Icon(Icons.add),
              label: const Text('Eigene Übung hinzufügen'),
              onPressed: () async {
                final nameController = TextEditingController();
                final muscleController = TextEditingController();
                await showDialog(context: context, builder: (ctx) => AlertDialog(
                  title: const Text('Neue Übung'),
                  content: Column(mainAxisSize: MainAxisSize.min, children: [
                    TextField(controller: nameController, decoration: const InputDecoration(labelText: 'Name')),
                    TextField(controller: muscleController, decoration: const InputDecoration(labelText: 'Muskelgruppe')),
                  ]),
                  actions: [
                    TextButton(onPressed: () => Navigator.pop(ctx), child: const Text('Abbruch')),
                    TextButton(onPressed: () async {
                      final n = nameController.text.trim();
                      final m = muscleController.text.trim();
                      if (n.isNotEmpty && m.isNotEmpty) {
                        await prov.addExercise(n, m);
                        Navigator.pop(ctx);
                      }
                    }, child: const Text('Speichern')),
                  ],
                ));
              },
            )
          ],
        ),
      ),
    );
  }
}
```

---

## lib/screens/workout_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/workout_provider.dart';

class WorkoutScreen extends StatefulWidget {
  final String day;
  final List<String> plannedExercises;
  const WorkoutScreen({super.key, required this.day, required this.plannedExercises});

  @override
  State<WorkoutScreen> createState() => _WorkoutScreenState();
}

class _WorkoutScreenState extends State<WorkoutScreen> {
  final Map<String, TextEditingController> _sets = {};
  final Map<String, TextEditingController> _reps = {};
  final Map<String, TextEditingController> _weight = {};

  @override
  void initState() {
    super.initState();
    for (final e in widget.plannedExercises) {
      _sets[e] = TextEditingController(text: '3');
      _reps[e] = TextEditingController(text: '8');
      _weight[e] = TextEditingController(text: '0');
    }
  }

  @override
  void dispose() {
    for (final c in _sets.values) c.dispose();
    for (final c in _reps.values) c.dispose();
    for (final c in _weight.values) c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);

    return Scaffold(
      appBar: AppBar(title: Text('${widget.day} - Training')),
      body: ListView(padding: const EdgeInsets.all(12), children: [
        for (final ex in widget.plannedExercises)
          Card(child: Padding(
            padding: const EdgeInsets.all(8.0),
            child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text(ex, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              Row(children: [
                Flexible(child: TextField(controller: _sets[ex], keyboardType: TextInputType.number, decoration: const InputDecoration(labelText: 'Sätze'))),
                const SizedBox(width: 8),
                Flexible(child: TextField(controller: _reps[ex], keyboardType: TextInputType.number, decoration: const InputDecoration(labelText: 'Wiederholungen'))),
                const SizedBox(width: 8),
                Flexible(child: TextField(controller: _weight[ex], keyboardType: const TextInputType.numberWithOptions(decimal: true), decoration: const InputDecoration(labelText: 'Gewicht (kg)'))),
              ]),
              const SizedBox(height: 8),
              Row(mainAxisAlignment: MainAxisAlignment.end, children: [
                ElevatedButton(
                  child: const Text('Speichern'),
                  onPressed: () async {
                    final sets = int.tryParse(_sets[ex]!.text) ?? 0;
                    final reps = int.tryParse(_reps[ex]!.text) ?? 0;
                    final weight = double.tryParse(_weight[ex]!.text) ?? 0.0;
                    if (sets <= 0 || reps <= 0) {
                      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Sätze und Wiederholungen müssen > 0 sein')));
                      return;
                    }
                    await prov.addEntry(ex, sets, reps, weight);
                    ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Eintrag gespeichert')));
                  },
                )
              ]),

              const SizedBox(height: 6),
              FutureBuilder(
                future: prov.lastEntriesForExercise(ex, limit: 3),
                builder: (context, snapshot) {
                  if (!snapshot.hasData) return const SizedBox();
                  final list = snapshot.data as List;
                  if (list.isEmpty) return const Text('Keine vorherigen Einträge', style: TextStyle(fontSize: 12));
                  return Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    const Divider(),
                    const Text('Letzte Einträge:', style: TextStyle(fontWeight: FontWeight.bold)),
                    ...list.map((e) => Text('${e.date}: ${e.sets}x${e.reps} @ ${e.weight}kg')),
                  ]);
                },
              )

            ]),
          )),
      ]),
    );
  }
}
```

---

## lib/screens/history_screen.dart

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/workout_provider.dart';

class HistoryScreen extends StatelessWidget {
  const HistoryScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);

    return Scaffold(
      appBar: AppBar(title: const Text('Training-Historie')),
      body: ListView.builder(
        itemCount: prov.history.length,
        itemBuilder: (ctx, i) {
          final e = prov.history[i];
          return ListTile(
            title: Text('${e.exercise} — ${e.sets}x${e.reps} @ ${e.weight}kg'),
            subtitle: Text(e.date),
          );
        },
      ),
    );
  }
}
```

---

## README.md (kurze Anleitung)

```
1. Flutter SDK installieren (https://flutter.dev)
2. Neues Projekt anlegen oder diese Dateien in ein bestehendes Flutter-Projekt kopieren.
3. pubspec.yaml übernehmen und `flutter pub get` ausführen.
4. Auf einem Gerät oder Emulator `flutter run` ausführen.

Erläuterungen:
- Die App speichert Daten lokal per sqflite (SQLite).
- Trainings-Entries speichern Datum, Übung, Sätze, Wiederholungen und Gewicht.
- Auf dem Workout-Screen werden die letzten Einträge pro Übung angezeigt, damit man beim nächsten Training Zahlen vergleichen kann.

Weiteres:
- Wenn du Sync/Cloud möchtest (z. B. Firebase), kann ich die erforderlichen Änderungen einbauen.
- Wenn du die App ins Play Store / App Store bringen willst, erkläre ich die Schritte zur Veröffentlichung.
```

---

Wenn du möchtest, kann ich:
- das Projekt als einzelne `main.dart`-Datei konsolidieren (für schnellen Test),
- oder das Projekt zu Flutter + Firebase erweitern (Cloud Sync + Auth),
- Tests hinzufügen oder UI-Details verbessern.

Sage kurz, welche Erweiterung du bevorzugst — ich liefere sofort den nächsten Code-Schritt.

---

## Konsolidierte single-file `main.dart` (für schnellen Test)

Füge die folgende Datei als `lib/main.dart` in ein neues Flutter-Projekt ein (pubspec.yaml wie im Starter-Projekt). Diese Variante enthält Modelle, DB-Init, Provider und die wichtigsten Screens in einer Datei, damit du schnell `flutter run` testen kannst.

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'package:intl/intl.dart';

// --- Models -----------------------------------------------------------------
class Exercise {
  final int? id;
  final String name;
  final String muscleGroup;
  Exercise({this.id, required this.name, required this.muscleGroup});
  Map<String, dynamic> toMap() => {'id': id, 'name': name, 'muscleGroup': muscleGroup};
  factory Exercise.fromMap(Map<String, dynamic> m) => Exercise(id: m['id'] as int?, name: m['name'] as String, muscleGroup: m['muscleGroup'] as String);
}

class WorkoutEntry {
  final int? id;
  final String date;
  final String exercise;
  final int sets;
  final int reps;
  final double weight;
  WorkoutEntry({this.id, required this.date, required this.exercise, required this.sets, required this.reps, required this.weight});
  Map<String, dynamic> toMap() => {'id': id, 'date': date, 'exercise': exercise, 'sets': sets, 'reps': reps, 'weight': weight};
  factory WorkoutEntry.fromMap(Map<String, dynamic> m) => WorkoutEntry(id: m['id'] as int?, date: m['date'] as String, exercise: m['exercise'] as String, sets: m['sets'] as int, reps: m['reps'] as int, weight: (m['weight'] as num).toDouble());
}

// --- Simple DB singleton ----------------------------------------------------
class AppDb {
  static final AppDb _i = AppDb._();
  factory AppDb() => _i;
  AppDb._();
  Database? _db;
  Future<Database> get database async {
    if (_db != null) return _db!;
    final path = join(await getDatabasesPath(), 'bodybuilding_single.db');
    _db = await openDatabase(path, version: 1, onCreate: (db, v) async {
      await db.execute('''CREATE TABLE exercises(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, muscleGroup TEXT NOT NULL)''');
      await db.execute('''CREATE TABLE entries(id INTEGER PRIMARY KEY AUTOINCREMENT, date TEXT NOT NULL, exercise TEXT NOT NULL, sets INTEGER NOT NULL, reps INTEGER NOT NULL, weight REAL NOT NULL)''');
      // seed
      final seed = [
        {'name':'Bankdrücken','muscle':'Brust'},
        {'name':'Schulterdrücken','muscle':'Schulter'},
        {'name':'Klimmzüge','muscle':'Latissimus'},
        {'name':'Kniebeuge','muscle':'Beine'},
        {'name':'Bizepscurls','muscle':'Bizeps'},
        {'name':'Trizepsdrücken','muscle':'Trizeps'},
      ];
      for (final s in seed) await db.insert('exercises', {'name': s['name'], 'muscleGroup': s['muscle']});
    });
    return _db!;
  }

  Future<List<Exercise>> getExercises() async {
    final d = await database;
    final res = await d.query('exercises', orderBy: 'name');
    return res.map((m) => Exercise.fromMap(m)).toList();
  }

  Future<int> insertExercise(Exercise e) async => (await database).insert('exercises', e.toMap());
  Future<int> insertEntry(WorkoutEntry we) async => (await database).insert('entries', we.toMap());
  Future<List<WorkoutEntry>> entriesForDate(String date) async {
    final res = await (await database).query('entries', where: 'date = ?', whereArgs: [date], orderBy: 'id DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }
  Future<List<WorkoutEntry>> entriesForExercise(String exercise) async {
    final res = await (await database).query('entries', where: 'exercise = ?', whereArgs: [exercise], orderBy: 'date DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }
  Future<List<WorkoutEntry>> allEntries() async {
    final res = await (await database).query('entries', orderBy: 'date DESC, id DESC');
    return res.map((m) => WorkoutEntry.fromMap(m)).toList();
  }
}

// --- Provider ---------------------------------------------------------------
class WorkoutProvider extends ChangeNotifier {
  final AppDb _db = AppDb();
  List<Exercise> exercises = [];
  List<WorkoutEntry> todayEntries = [];
  List<WorkoutEntry> history = [];

  Future<void> init() async {
    exercises = await _db.getExercises();
    await loadToday();
    history = await _db.allEntries();
    notifyListeners();
  }

  String todayISO() => DateFormat('yyyy-MM-dd').format(DateTime.now());

  Future<void> loadToday() async {
    todayEntries = await _db.entriesForDate(todayISO());
    notifyListeners();
  }

  Future<void> addExercise(String name, String muscle) async {
    await _db.insertExercise(Exercise(name: name, muscleGroup: muscle));
    exercises = await _db.getExercises();
    notifyListeners();
  }

  Future<void> addEntry(String exercise, int sets, int reps, double weight) async {
    final e = WorkoutEntry(date: todayISO(), exercise: exercise, sets: sets, reps: reps, weight: weight);
    await _db.insertEntry(e);
    await loadToday();
    history = await _db.allEntries();
    notifyListeners();
  }

  Future<List<WorkoutEntry>> lastEntriesForExercise(String exercise, {int limit = 3}) async {
    final list = await _db.entriesForExercise(exercise);
    return list.take(limit).toList();
  }
}

// --- App -------------------------------------------------------------------
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(create: (_) => WorkoutProvider()..init(), child: MaterialApp(title: 'Bodybuilding Tracker', theme: ThemeData(primarySwatch: Colors.blue), home: const HomeScreen()));
  }
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);
    final split = {
      'Montag': ['Bankdrücken', 'Schulterdrücken', 'Trizepsdrücken'],
      'Dienstag': ['Klimmzüge', 'Bizepscurls', 'Kniebeuge'],
      'Mittwoch': ['Pause'],
      'Donnerstag': ['Bankdrücken', 'Rudern', 'Beinpresse'],
      'Freitag': ['Schulterdrücken', 'Seitheben', 'Trizepsdrücken'],
    };

    return Scaffold(
      appBar: AppBar(title: const Text('Bodybuilding Tracker'), actions: [IconButton(icon: const Icon(Icons.history), onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const HistoryScreen())))]),
      body: RefreshIndicator(
        onRefresh: prov.init,
        child: ListView(padding: const EdgeInsets.all(12), children: [
          const Text('Trainings-Split (Beispiel)', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          ...split.entries.map((e) => Card(child: ListTile(title: Text(e.key), subtitle: Text(e.value.join(', ')), onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => WorkoutScreen(day: e.key, plannedExercises: e.value.toList())))))),
          const SizedBox(height: 16),
          const Text('Übungsbibliothek', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          ...prov.exercises.map((ex) => ListTile(title: Text(ex.name), subtitle: Text(ex.muscleGroup), trailing: const Icon(Icons.chevron_right), onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => WorkoutScreen(day: 'Freies Training', plannedExercises: [ex.name]))))),
          const SizedBox(height: 12),
          ElevatedButton.icon(icon: const Icon(Icons.add), label: const Text('Eigene Übung hinzufügen'), onPressed: () async {
            final nameController = TextEditingController();
            final muscleController = TextEditingController();
            await showDialog(context: context, builder: (ctx) => AlertDialog(title: const Text('Neue Übung'), content: Column(mainAxisSize: MainAxisSize.min, children: [TextField(controller: nameController, decoration: const InputDecoration(labelText: 'Name')), TextField(controller: muscleController, decoration: const InputDecoration(labelText: 'Muskelgruppe'))]), actions: [TextButton(onPressed: () => Navigator.pop(ctx), child: const Text('Abbruch')), TextButton(onPressed: () async {final n = nameController.text.trim(); final m = muscleController.text.trim(); if (n.isNotEmpty && m.isNotEmpty) { await Provider.of<WorkoutProvider>(context, listen: false).addExercise(n, m); Navigator.pop(ctx); }}, child: const Text('Speichern'))]));
          })
        ]),
      ),
    );
  }
}

class WorkoutScreen extends StatefulWidget {
  final String day;
  final List<String> plannedExercises;
  const WorkoutScreen({super.key, required this.day, required this.plannedExercises});
  @override
  State<WorkoutScreen> createState() => _WorkoutScreenState();
}

class _WorkoutScreenState extends State<WorkoutScreen> {
  final Map<String, TextEditingController> sets = {};
  final Map<String, TextEditingController> reps = {};
  final Map<String, TextEditingController> weight = {};

  @override
  void initState() {
    super.initState();
    for (final e in widget.plannedExercises) {
      sets[e] = TextEditingController(text: '3');
      reps[e] = TextEditingController(text: '8');
      weight[e] = TextEditingController(text: '0');
    }
  }

  @override
  void dispose() { for (final c in sets.values) c.dispose(); for (final c in reps.values) c.dispose(); for (final c in weight.values) c.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);
    return Scaffold(appBar: AppBar(title: Text('${widget.day} - Training')), body: ListView(padding: const EdgeInsets.all(12), children: [
      for (final ex in widget.plannedExercises)
        Card(child: Padding(padding: const EdgeInsets.all(8.0), child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
          Text(ex, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)), const SizedBox(height: 8),
          Row(children: [Flexible(child: TextField(controller: sets[ex], keyboardType: TextInputType.number, decoration: const InputDecoration(labelText: 'Sätze'))), const SizedBox(width: 8), Flexible(child: TextField(controller: reps[ex], keyboardType: TextInputType.number, decoration: const InputDecoration(labelText: 'Wiederholungen'))), const SizedBox(width: 8), Flexible(child: TextField(controller: weight[ex], keyboardType: const TextInputType.numberWithOptions(decimal: true), decoration: const InputDecoration(labelText: 'Gewicht (kg)'))),]), const SizedBox(height: 8), Row(mainAxisAlignment: MainAxisAlignment.end, children: [ElevatedButton(child: const Text('Speichern'), onPressed: () async { final s = int.tryParse(sets[ex]!.text) ?? 0; final r = int.tryParse(reps[ex]!.text) ?? 0; final w = double.tryParse(weight[ex]!.text) ?? 0.0; if (s<=0||r<=0) { ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Sätze und Wiederholungen müssen > 0 sein'))); return; } await prov.addEntry(ex, s, r, w); ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Eintrag gespeichert'))); })]), const SizedBox(height: 6), FutureBuilder(future: prov.lastEntriesForExercise(ex, limit: 3), builder: (context, snapshot) { if (!snapshot.hasData) return const SizedBox(); final list = snapshot.data as List; if (list.isEmpty) return const Text('Keine vorherigen Einträge', style: TextStyle(fontSize: 12)); return Column(crossAxisAlignment: CrossAxisAlignment.start, children: [ const Divider(), const Text('Letzte Einträge:', style: TextStyle(fontWeight: FontWeight.bold)), ...list.map((e) => Text('${e.date}: ${e.sets}x${e.reps} @ ${e.weight}kg')) ]); }) ]))
    ]));
  }
}

class HistoryScreen extends StatelessWidget {
  const HistoryScreen({super.key});
  @override
  Widget build(BuildContext context) {
    final prov = Provider.of<WorkoutProvider>(context);
    return Scaffold(appBar: AppBar(title: const Text('Training-Historie')), body: ListView.builder(itemCount: prov.history.length, itemBuilder: (ctx, i) { final e = prov.history[i]; return ListTile(title: Text('${e.exercise} — ${e.sets}x${e.reps} @ ${e.weight}kg'), subtitle: Text(e.date)); }));
  }
}
```

Hinweis: diese single-file-Version ist bewusst pragmatisch gehalten, um schnellen Test zu ermöglichen. Für Produktion empfehle ich modulare Struktur (Modelle, DB, Provider in separaten Dateien) und Tests.

---
