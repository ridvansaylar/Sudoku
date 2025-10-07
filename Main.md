// main.dart - Tam entegre sürüm
import 'dart:math';
import 'package:flutter/material.dart';

void main() {
  runApp(const SudokuApp());
}

/* --------------------
   Basit Sudoku Modeli
   (EditorScreen tarafından kullanılan sınıflar)
   -------------------- */

class Cell {
  int? base; // kilitli temel sayı (Başla sonrası)
  int? tempBase; // Oluşturma safhasında girilen (başla öncesi)
  int? user; // oyuncunun girdiği tam sayı
  Set<int> centerCandidates = {}; // merkez adayları (max 4)
  List<int> cornerCandidates = []; // köşe adayları (max 4, sıralı)
}

class MoveSnapshot {
  final int row;
  final int col;
  final int? prevUser;
  final Set<int> prevCenter;
  final List<int> prevCorner;
  final int? prevTempBase;

  MoveSnapshot({
    required this.row,
    required this.col,
    this.prevUser,
    Set<int>? prevCenter,
    List<int>? prevCorner,
    this.prevTempBase,
  })  : prevCenter = prevCenter == null ? {} : Set.from(prevCenter),
        prevCorner = prevCorner == null ? [] : List.from(prevCorner);
}

/* --------------------
   App & UI
   -------------------- */

class SudokuApp extends StatelessWidget {
  const SudokuApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Sudoku App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: false),
      home: const MainMenu(),
    );
  }
}

/* --------------------
   Main Menu (Oyna, Oluştur, Ayarlar)
   -------------------- */

class MainMenu extends StatelessWidget {
  const MainMenu({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Sudoku Uygulaması")),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: 28.0),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              ElevatedButton(
                onPressed: () {
                  Navigator.push(context, MaterialPageRoute(builder: (_) => const DifficultySelectionPage()));
                },
                child: const Padding(
                  padding: EdgeInsets.symmetric(horizontal: 36, vertical: 14),
                  child: Text("Oyna", style: TextStyle(fontSize: 20)),
                ),
              ),
              const SizedBox(height: 14),
              ElevatedButton(
                onPressed: () {
                  // "Oluştur" zaten senin EditorScreen'in
                  Navigator.push(context, MaterialPageRoute(builder: (_) => const EditorScreen()));
                },
                child: const Padding(
                  padding: EdgeInsets.symmetric(horizontal: 24, vertical: 12),
                  child: Text("Oluştur", style: TextStyle(fontSize: 18)),
                ),
              ),
              const SizedBox(height: 14),
              ElevatedButton(
                onPressed: () {
                  // Ayarlar ekranı ileride genişletilecek
                  showDialog(
                    context: context,
                    builder: (_) => AlertDialog(
                      title: const Text("Ayarlar"),
                      content: const Text("Ayarlar ekranı henüz hazırlanmadı."),
                      actions: [TextButton(onPressed: () => Navigator.pop(context), child: const Text("Kapat"))],
                    ),
                  );
                },
                child: const Padding(
                  padding: EdgeInsets.symmetric(horizontal: 24, vertical: 12),
                  child: Text("Ayarlar", style: TextStyle(fontSize: 18)),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

/* --------------------
   Difficulty Selection Page
   -------------------- */

class DifficultySelectionPage extends StatelessWidget {
  const DifficultySelectionPage({super.key});

  void _startGame(BuildContext context, String difficulty) {
    // üretici kullanarak puzzle ve solution oluştur
    final gen = SudokuGenerator();
    final solved = gen.generateSolvedGrid();
    final puzzle = gen.generatePuzzle(solved, _emptyCellsForDifficulty(difficulty));
    Navigator.push(
      context,
      MaterialPageRoute(builder: (_) => SudokuGamePage(puzzle: puzzle, solution: solved, difficulty: difficulty)),
    );
  }

  static int _emptyCellsForDifficulty(String d) {
    switch (d) {
      case 'Kolay':
        return 40;
      case 'Orta':
        return 50;
      case 'Zor':
        return 60;
      default:
        return 50;
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Zorluk Seçimi")),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ElevatedButton(onPressed: () => _startGame(context, "Kolay"), child: const Text("Kolay")),
            const SizedBox(height: 8),
            ElevatedButton(onPressed: () => _startGame(context, "Orta"), child: const Text("Orta")),
            const SizedBox(height: 8),
            ElevatedButton(onPressed: () => _startGame(context, "Zor"), child: const Text("Zor")),
          ],
        ),
      ),
    );
  }
}

/* --------------------
   Sudoku Generator (Backtracking)
   -------------------- */

class SudokuGenerator {
  final Random _rand = Random();

  List<List<int>> generateSolvedGrid() {
    List<List<int>> grid = List.generate(9, (_) => List.filled(9, 0));
    _fillGrid(grid);
    return grid;
  }

  bool _fillGrid(List<List<int>> grid) {
    for (int row = 0; row < 9; row++) {
      for (int col = 0; col < 9; col++) {
        if (grid[row][col] == 0) {
          List<int> nums = List.generate(9, (i) => i + 1)..shuffle(_rand);
          for (int n in nums) {
            if (_isSafe(grid, row, col, n)) {
              grid[row][col] = n;
              if (_fillGrid(grid)) return true;
              grid[row][col] = 0;
            }
          }
          return false;
        }
      }
    }
    return true;
  }

  bool _isSafe(List<List<int>> grid, int row, int col, int num) {
    for (int i = 0; i < 9; i++) {
      if (grid[row][i] == num || grid[i][col] == num) return false;
    }
    int startRow = row - row % 3, startCol = col - col % 3;
    for (int r = 0; r < 3; r++) {
      for (int c = 0; c < 3; c++) {
        if (grid[startRow + r][startCol + c] == num) return false;
      }
    }
    return true;
  }

  List<List<int>> generatePuzzle(List<List<int>> solved, int emptyCells) {
    List<List<int>> puzzle = solved.map((r) => List<int>.from(r)).toList();
    int removed = 0;
    while (removed < emptyCells) {
      int r = _rand.nextInt(9);
      int c = _rand.nextInt(9);
      if (puzzle[r][c] != 0) {
        puzzle[r][c] = 0;
        removed++;
      }
    }
    return puzzle;
  }
}

/* --------------------
   Sudoku Game Page (Oyna)
   -------------------- */

class SudokuGamePage extends StatefulWidget {
  final List<List<int>> puzzle;
  final List<List<int>> solution;
  final String difficulty;

  const SudokuGamePage({
    super.key,
    required this.puzzle,
    required this.solution,
    required this.difficulty,
  });

  @override
  State<SudokuGamePage> createState() => _SudokuGamePageState();
}

class _SudokuGamePageState extends State<SudokuGamePage> {
  late List<List<int>> board;
  int? selectedRow;
  int? selectedCol;

  // Modlar
  bool writeMode = true; // varsayılan
  bool centerMode = false;
  bool cornerMode = false;

  // Sabit hücreler
  final List<List<bool>> given =
      List.generate(9, (_) => List.generate(9, (_) => false));

  // Aday listeleri
  final List<List<Set<int>>> centerCandidates =
      List.generate(9, (_) => List.generate(9, (_) => <int>{}));
  final List<List<List<int>>> cornerCandidates =
      List.generate(9, (_) => List.generate(9, (_) => <int>[]));

  // 🔙 Geri al sistemi
  final List<List<List<int>>> history = [];

  @override
  void initState() {
    super.initState();
    board = widget.puzzle.map((r) => List<int>.from(r)).toList();
    for (int r = 0; r < 9; r++) {
      for (int c = 0; c < 9; c++) {
        given[r][c] = widget.puzzle[r][c] != 0;
      }
    }
    saveState(); // başlangıç durumu
  }

  void saveState() {
    history.add(board.map((r) => List<int>.from(r)).toList());
  }

  void undo() {
    if (history.length <= 1) return; // ilk durumu koru
    setState(() {
      history.removeLast();
      board = history.last.map((r) => List<int>.from(r)).toList();
    });
  }

  void selectCell(int r, int c) {
    if (given[r][c]) return;
    setState(() {
      selectedRow = r;
      selectedCol = c;
    });
  }

  void insertNumber(int n) {
    if (selectedRow == null || selectedCol == null) return;
    final r = selectedRow!;
    final c = selectedCol!;

    setState(() {
      if (writeMode) {
        board[r][c] = n;
        centerCandidates[r][c].clear();
        cornerCandidates[r][c].clear();
        saveState();
      } else if (centerMode) {
        if (centerCandidates[r][c].contains(n)) {
          centerCandidates[r][c].remove(n);
        } else {
          if (centerCandidates[r][c].length >= 4) return;
          centerCandidates[r][c].add(n);
        }
      } else if (cornerMode) {
        if (cornerCandidates[r][c].contains(n)) {
          cornerCandidates[r][c].remove(n);
        } else {
          if (cornerCandidates[r][c].length >= 4) return;
          cornerCandidates[r][c].add(n);
        }
      }
    });
  }

  void deleteNumber() {
    if (selectedRow == null || selectedCol == null) return;
    final r = selectedRow!;
    final c = selectedCol!;
    setState(() {
      board[r][c] = 0;
      centerCandidates[r][c].clear();
      cornerCandidates[r][c].clear();
      saveState();
    });
  }

  bool isConflict(int row, int col, int val) {
    if (val == 0) return false;

    // Satır kontrol
    for (int c = 0; c < 9; c++) {
      if (c != col && board[row][c] == val) return true;
    }

    // Sütun kontrol
    for (int r = 0; r < 9; r++) {
      if (r != row && board[r][col] == val) return true;
    }

    // 3x3 blok kontrol
    int startRow = (row ~/ 3) * 3;
    int startCol = (col ~/ 3) * 3;
    for (int r = startRow; r < startRow + 3; r++) {
      for (int c = startCol; c < startCol + 3; c++) {
        if (!(r == row && c == col) && board[r][c] == val) return true;
      }
    }

    return false;
  }

  bool isCompleteCorrect() {
    for (int r = 0; r < 9; r++) {
      for (int c = 0; c < 9; c++) {
        if (board[r][c] != widget.solution[r][c]) return false;
      }
    }
    return true;
  }

  Widget buildCell(int index) {
    int row = index ~/ 9;
    int col = index % 9;
    bool isGiven = given[row][col];
    bool isSelected = selectedRow == row && selectedCol == col;
    bool highlightRow = selectedRow != null && selectedRow == row;
    bool highlightCol = selectedCol != null && selectedCol == col;

    final val = board[row][col];
    final centers = centerCandidates[row][col];
    final corners = cornerCandidates[row][col];

    bool conflict = isConflict(row, col, val);

    BorderSide thick = const BorderSide(width: 2, color: Colors.black);
    BorderSide thin = const BorderSide(width: 0.5, color: Colors.black26);

    return GestureDetector(
      onTap: () => selectCell(row, col),
      child: Container(
        decoration: BoxDecoration(
          color: isSelected
              ? Colors.blue.withOpacity(0.15)
              : (highlightRow || highlightCol
                  ? Colors.grey.withOpacity(0.08)
                  : Colors.white),
          border: Border(
            top: row % 3 == 0 ? thick : thin,
            left: col % 3 == 0 ? thick : thin,
            right: (col + 1) % 3 == 0 ? thick : thin,
            bottom: (row + 1) % 3 == 0 ? thick : thin,
          ),
        ),
        child: Stack(
          children: [
            if (val != 0)
              Center(
                child: Text(
                  "$val",
                  style: TextStyle(
                    fontSize: 22,
                    fontWeight: isGiven ? FontWeight.bold : FontWeight.normal,
                    color: isGiven
                        ? Colors.black87
                        : (conflict ? Colors.red : Colors.black),
                  ),
                ),
              ),
            // Merkez küçük adaylar
            if (centers.isNotEmpty)
              Center(
                child: Wrap(
                  alignment: WrapAlignment.center,
                  spacing: 2,
                  runSpacing: 2,
                  children: centers
                      .map((n) => Text(
                            "$n",
                            style: const TextStyle(
                              fontSize: 10,
                              color: Colors.grey,
                            ),
                          ))
                      .toList(),
                ),
              ),
            // Köşe küçük adaylar
            if (corners.isNotEmpty)
              Padding(
                padding: const EdgeInsets.all(3),
                child: Align(
                  alignment: Alignment.topLeft,
                  child: Wrap(
                    spacing: 1,
                    runSpacing: 1,
                    children: corners
                        .map((n) => Text(
                              "$n",
                              style: const TextStyle(
                                fontSize: 9,
                                color: Colors.grey,
                              ),
                            ))
                        .toList(),
                  ),
                ),
              ),
          ],
        ),
      ),
    );
  }

  Widget numButton(int n) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 4),
      child: SizedBox(
        width: 52,
        height: 52,
        child: ElevatedButton(
          style: ElevatedButton.styleFrom(
            padding: EdgeInsets.zero,
            shape:
                const RoundedRectangleBorder(borderRadius: BorderRadius.zero),
            backgroundColor: Colors.white,
            foregroundColor: Colors.black,
            side: const BorderSide(color: Colors.black12),
          ),
          onPressed: () => insertNumber(n),
          child: Text("$n", style: const TextStyle(fontSize: 20)),
        ),
      ),
    );
  }

  Widget modeButton(String text, bool active, VoidCallback onTap) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 6),
      child: ElevatedButton(
        style: ElevatedButton.styleFrom(
          backgroundColor: active ? Colors.blue : Colors.white,
          foregroundColor: active ? Colors.white : Colors.black,
          side: const BorderSide(color: Colors.black12),
        ),
        onPressed: onTap,
        child: Text(text),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Sudoku - ${widget.difficulty}")),
      body: SafeArea(
        child: Column(
          children: [
            const SizedBox(height: 8),
            // 🔵 Mod butonları
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                modeButton("Merkez", centerMode, () {
                  setState(() {
                    centerMode = true;
                    cornerMode = false;
                    writeMode = false;
                  });
                }),
                modeButton("Köşe", cornerMode, () {
                  setState(() {
                    cornerMode = true;
                    centerMode = false;
                    writeMode = false;
                  });
                }),
                modeButton("Yaz", writeMode, () {
                  setState(() {
                    writeMode = true;
                    centerMode = false;
                    cornerMode = false;
                  });
                }),
              ],
            ),
            const SizedBox(height: 8),

            // Sudoku ızgarası
            Expanded(
              flex: 6,
              child: Padding(
                padding: const EdgeInsets.all(12.0),
                child: AspectRatio(
                  aspectRatio: 1,
                  child: GridView.builder(
                    physics: const NeverScrollableScrollPhysics(),
                    gridDelegate:
                        const SliverGridDelegateWithFixedCrossAxisCount(
                            crossAxisCount: 9),
                    itemCount: 81,
                    itemBuilder: (context, index) {
                      return buildCell(index);
                    },
                  ),
                ),
              ),
            ),

            // Numpad
            Expanded(
              flex: 3,
              child: Column(
                children: [
                  const SizedBox(height: 6),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: List.generate(5, (i) => numButton(i + 1)),
                  ),
                  const SizedBox(height: 8),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: List.generate(4, (i) => numButton(i + 6)),
                  ),
                  const SizedBox(height: 12),

                  // Alt butonlar
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                    children: [
                      ElevatedButton.icon(
                        onPressed: deleteNumber,
                        icon: const Icon(Icons.backspace_outlined),
                        label: const Text("Sil"),
                      ),
                      ElevatedButton.icon(
                        onPressed: () {
                          if (isCompleteCorrect()) {
                            showDialog(
                              context: context,
                              builder: (_) => AlertDialog(
                                title: const Text("Tebrikler!"),
                                content:
                                    const Text("Bulmacayı doğru çözdünüz."),
                                actions: [
                                  TextButton(
                                      onPressed: () => Navigator.pop(context),
                                      child: const Text("Tamam"))
                                ],
                              ),
                            );
                          } else {
                            ScaffoldMessenger.of(context).showSnackBar(
                              const SnackBar(content: Text("Henüz doğru değil.")),
                            );
                          }
                        },
                        icon: const Icon(Icons.check),
                        label: const Text("Kontrol Et"),
                      ),
                      ElevatedButton.icon(
                        onPressed: undo,
                        icon: const Icon(Icons.undo),
                        label: const Text("Geri Al"),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}




/* --------------------
   Editor / Oluştur Ekranı
   --------------------
   Burada senin daha önce gönderdiğin uzun EditorScreen kodu birebir korunmuştur.
   Aşağıya senin gönderdiğin EditorScreen içeriğini tam olarak ekledim.
   (Kodun tamamen aynısını buraya yerleştirdim)
   -------------------- */

class EditorScreen extends StatefulWidget {
  const EditorScreen({super.key});

  @override
  State<EditorScreen> createState() => _EditorScreenState();
}

class _EditorScreenState extends State<EditorScreen> {
  final List<List<Cell>> board = List.generate(9, (_) => List.generate(9, (_) => Cell()));
  int? selectedRow;
  int? selectedCol;

  bool started = false;
  bool writeMode = false;
  bool centerMode = false;
  bool cornerMode = false;

  final List<MoveSnapshot> undoStack = [];

  // Seçili hücreyi belirle
  void selectCell(int r, int c) {
    setState(() {
      selectedRow = r;
      selectedCol = c;
    });
  }

  // Helper: snapshot push
  void pushSnapshot(int r, int c) {
    final Cell cell = board[r][c];
    undoStack.add(MoveSnapshot(
      row: r,
      col: c,
      prevUser: cell.user,
      prevCenter: Set.from(cell.centerCandidates),
      prevCorner: List.from(cell.cornerCandidates),
      prevTempBase: cell.tempBase,
    ));
  }

  // Numpad'e basıldı (1-9)
  void onNumberPressed(int number) {
    if (selectedRow == null || selectedCol == null) return;
    final r = selectedRow!;
    final c = selectedCol!;
    final cell = board[r][c];

    if (!started) {
      // Oluşturma safhası: tempBase atıyoruz (dilediğin gibi düzenle)
      setState(() {
        cell.tempBase = number;
      });
      return;
    }

    // Başla sonrası: çözme modu
    if (writeMode) {
      // Yaz modu: tek rakam girer (önceki user kaydedilir)
      pushSnapshot(r, c);
      setState(() {
        cell.user = number;
        // kullanıcı tam sayı girince adayları temizleyebiliriz
        cell.centerCandidates.clear();
        cell.cornerCandidates.clear();
      });
    } else if (centerMode) {
      // merkez aday toggle, max 4
      if (cell.centerCandidates.contains(number)) {
        pushSnapshot(r, c);
        setState(() {
          cell.centerCandidates.remove(number);
        });
      } else {
        if (cell.centerCandidates.length >= 4) return; // en fazla 4
        pushSnapshot(r, c);
        setState(() {
          cell.centerCandidates.add(number);
        });
      }
    } else if (cornerMode) {
      // köşe aday toggle (sıralı liste)
      if (cell.cornerCandidates.contains(number)) {
        pushSnapshot(r, c);
        setState(() {
          cell.cornerCandidates.remove(number);
        });
      } else {
        if (cell.cornerCandidates.length >= 4) return;
        pushSnapshot(r, c);
        setState(() {
          cell.cornerCandidates.add(number);
        });
      }
    } else {
      // Hiçbir mod açıksa, varsayılan "Yaz" gibi davranabiliriz:
      pushSnapshot(r, c);
      setState(() {
        cell.user = number;
      });
    }
  }

  // Silme
  void onDeletePressed() {
    if (selectedRow == null || selectedCol == null) return;
    final r = selectedRow!;
    final c = selectedCol!;
    final cell = board[r][c];

    // base hücresiyse silinemez
    if (cell.base != null) return;

    // snapshot kaydet
    pushSnapshot(r, c);
    setState(() {
      cell.user = null;
      cell.centerCandidates.clear();
      cell.cornerCandidates.clear();
      // Oluşturma öncesi tempBase varsa ve henüz started==false ise de silinebilir
      if (!started) cell.tempBase = null;
    });
  }

  // Undo
  void onUndoPressed() {
    if (undoStack.isEmpty) return;
    final snapshot = undoStack.removeLast();
    setState(() {
      final cell = board[snapshot.row][snapshot.col];
      cell.user = snapshot.prevUser;
      cell.centerCandidates = Set.from(snapshot.prevCenter);
      cell.cornerCandidates = List.from(snapshot.prevCorner);
      cell.tempBase = snapshot.prevTempBase;
    });
  }

  // Başla (başlangıçta Oluştur sonrası çağırılacak)
  void onStartPressed() {
    setState(() {
      // tempBase -> base
      for (int r = 0; r < 9; r++) {
        for (int c = 0; c < 9; c++) {
          final cell = board[r][c];
          if (cell.tempBase != null) {
            cell.base = cell.tempBase;
            cell.tempBase = null;
          }
        }
      }
      started = true;
      // modları resetle
      writeMode = false;
      centerMode = false;
      cornerMode = false;
      undoStack.clear();
    });
  }

  // Oluşturma ekrandayken "Başla" değil de "İptal" gibi davranırsan önceki değerler korunur
  // Yaz / Merkez / Köşe toggle mantığı:
  void toggleWriteMode() {
    setState(() {
      writeMode = !writeMode;
      if (writeMode) {
        centerMode = false;
        cornerMode = false;
      }
    });
  }

  void toggleCenterMode() {
    setState(() {
      centerMode = !centerMode;
      if (centerMode) {
        writeMode = false;
        cornerMode = false;
      }
    });
  }

  void toggleCornerMode() {
    setState(() {
      cornerMode = !cornerMode;
      if (cornerMode) {
        writeMode = false;
        centerMode = false;
      }
    });
  }

  // Hücre widget'ı
  Widget buildCell(int index) {
    final row = index ~/ 9;
    final col = index % 9;
    final cell = board[row][col];
    final bool isSelected = (selectedRow == row && selectedCol == col);
    final bool highlightRow = selectedRow != null && row == selectedRow;
    final bool highlightCol = selectedCol != null && col == selectedCol;

    final baseVal = cell.base;
    final tempVal = cell.tempBase;
    final userVal = cell.user;
    final centerList = cell.centerCandidates.toList()..sort();
    final cornerList = cell.cornerCandidates;

    return GestureDetector(
      onTap: () {
        selectCell(row, col);
      },
      child: LayoutBuilder(builder: (context, constraints) {
        final double fontSizeMain = constraints.biggest.width * 0.5;
        final double smallFont = constraints.biggest.width * 0.22;
        return Container(
          color: isSelected
              ? Colors.blue.withOpacity(0.15)
              : (highlightRow || highlightCol)
              ? Colors.grey.withOpacity(0.08)
              : null,
          child: Stack(
            children: [
              Container(
                decoration: BoxDecoration(
                  border: Border(
                    top: BorderSide(width: row % 3 == 0 ? 2 : 0.5, color: Colors.black),
                    left: BorderSide(width: col % 3 == 0 ? 2 : 0.5, color: Colors.black),
                    right: BorderSide(width: (col + 1) % 3 == 0 ? 2 : 0.5, color: Colors.black),
                    bottom: BorderSide(width: (row + 1) % 3 == 0 ? 2 : 0.5, color: Colors.black),
                  ),
                ),
              ),

              Positioned.fill(
                child: Padding(
                  padding: const EdgeInsets.all(2.0),
                  child: Stack(
                    children: [
                      if (baseVal != null)
                        Align(
                          alignment: Alignment.center,
                          child: Opacity(
                            opacity: 0.45,
                            child: Text(
                              baseVal.toString(),
                              style: TextStyle(fontSize: fontSizeMain, fontWeight: FontWeight.bold, color: Colors.black),
                            ),
                          ),
                        ),
                      if (userVal != null)
                        Align(
                          alignment: Alignment.center,
                          child: Text(
                            userVal.toString(),
                            style: TextStyle(fontSize: fontSizeMain, fontWeight: FontWeight.w700, color: Colors.black),
                          ),
                        ),
                      if (centerList.isNotEmpty)
                        Align(
                          alignment: Alignment.center,
                          child: Wrap(
                            alignment: WrapAlignment.center,
                            spacing: 2,
                            runSpacing: 0,
                            children: centerList.map((n) {
                              return Text(
                                n.toString(),
                                style: TextStyle(fontSize: smallFont, color: Colors.black87),
                              );
                            }).toList(),
                          ),
                        ),
                      if (cornerList.isNotEmpty)
                        Positioned(
                          left: 0,
                          top: 0,
                          right: 0,
                          bottom: 0,
                          child: IgnorePointer(
                            child: Stack(
                              children: [
                                if (cornerList.length >= 1)
                                  Positioned(left: 2, top: 0, child: Text(cornerList[0].toString(), style: TextStyle(fontSize: smallFont))),
                                if (cornerList.length >= 2)
                                  Positioned(right: 2, top: 0, child: Text(cornerList[1].toString(), style: TextStyle(fontSize: smallFont))),
                                if (cornerList.length >= 3)
                                  Positioned(left: 2, bottom: 0, child: Text(cornerList[2].toString(), style: TextStyle(fontSize: smallFont))),
                                if (cornerList.length >= 4)
                                  Positioned(right: 2, bottom: 0, child: Text(cornerList[3].toString(), style: TextStyle(fontSize: smallFont))),
                              ],
                            ),
                          ),
                        ),
                      if (!started && tempVal != null)
                        Align(
                          alignment: Alignment.center,
                          child: Opacity(
                            opacity: 0.65,
                            child: Text(
                              tempVal.toString(),
                              style: TextStyle(fontSize: fontSizeMain, fontWeight: FontWeight.bold, color: Colors.black54),
                            ),
                          ),
                        ),
                    ],
                  ),
                ),
              ),
            ],
          ),
        );
      }),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Oluştur / Çöz"),
        actions: [
          if (!started)
            TextButton(
              onPressed: onStartPressed,
              child: const Text("Başla", style: TextStyle(color: Colors.white, fontSize: 20, fontWeight: FontWeight.bold)),
            ),
          if (started)
            IconButton(
              onPressed: () {
                setState(() {
                  for (var row in board) {
                    for (var cell in row) {
                      cell.base = null;
                      cell.tempBase = null;
                      cell.user = null;
                      cell.centerCandidates.clear();
                      cell.cornerCandidates.clear();
                    }
                  }
                  started = false;
                  writeMode = false;
                  centerMode = false;
                  cornerMode = false;
                  undoStack.clear();
                  selectedRow = null;
                  selectedCol = null;
                });
              },
              icon: const Icon(Icons.restart_alt),
              tooltip: "Yeniden Oluştur",
            ),
        ],
      ),
      body: SafeArea(
        child: Column(
          children: [
            const SizedBox(height: 8),
            Expanded(
              flex: 6,
              child: Center(
                child: Padding(
                  padding: const EdgeInsets.all(12.0),
                  child: AspectRatio(
                    aspectRatio: 1,
                    child: GridView.builder(
                      physics: const NeverScrollableScrollPhysics(),
                      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 9),
                      itemCount: 81,
                      itemBuilder: (context, index) {
                        return buildCell(index);
                      },
                    ),
                  ),
                ),
              ),
            ),
            Expanded(
              flex: 3,
              child: Column(
                children: [
                  Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceAround,
                      children: [
                        ToggleButtonWidget(label: "Yaz", active: writeMode, onTap: toggleWriteMode),
                        ToggleButtonWidget(label: "Merkez", active: centerMode, onTap: toggleCenterMode),
                        ToggleButtonWidget(label: "Köşe", active: cornerMode, onTap: toggleCornerMode),
                        ElevatedButton.icon(onPressed: onUndoPressed, icon: const Icon(Icons.undo), label: const Text("Geri Al")),
                        ElevatedButton.icon(onPressed: onDeletePressed, icon: const Icon(Icons.backspace_outlined), label: const Text("Sil")),
                      ],
                    ),
                  ),
                  const SizedBox(height: 8),
                  // numpad iki satır
                  Column(
                    children: [
                      Row(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: List.generate(5, (i) {
                          return Padding(
                            padding: const EdgeInsets.symmetric(horizontal: 4),
                            child: SizedBox(
                              width: 44,
                              height: 44,
                              child: ElevatedButton(
                                style: ElevatedButton.styleFrom(
                                  padding: EdgeInsets.zero,
                                  shape: const RoundedRectangleBorder(borderRadius: BorderRadius.zero),
                                  backgroundColor: Colors.white,
                                  foregroundColor: Colors.black,
                                  side: const BorderSide(color: Colors.black12),
                                ),
                                onPressed: () => onNumberPressed(i + 1),
                                child: Text("${i + 1}", style: const TextStyle(fontSize: 18)),
                              ),
                            ),
                          );
                        }),
                      ),
                      const SizedBox(height: 8),
                      Row(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: List.generate(4, (i) {
                          int num = i + 6;
                          return Padding(
                            padding: const EdgeInsets.symmetric(horizontal: 4),
                            child: SizedBox(
                              width: 44,
                              height: 44,
                              child: ElevatedButton(
                                style: ElevatedButton.styleFrom(
                                  padding: EdgeInsets.zero,
                                  shape: const RoundedRectangleBorder(borderRadius: BorderRadius.zero),
                                  backgroundColor: Colors.white,
                                  foregroundColor: Colors.black,
                                  side: const BorderSide(color: Colors.black12),
                                ),
                                onPressed: () => onNumberPressed(num),
                                child: Text("$num", style: const TextStyle(fontSize: 18)),
                              ),
                            ),
                          );
                        }),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

/* --------------------
   Toggle small button
   -------------------- */

class ToggleButtonWidget extends StatelessWidget {
  final String label;
  final bool active;
  final VoidCallback onTap;
  const ToggleButtonWidget({required this.label, required this.active, required this.onTap, super.key});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 8),
        decoration: BoxDecoration(
          color: active ? Colors.green : Colors.grey[200],
          borderRadius: BorderRadius.circular(8),
          border: Border.all(color: Colors.black12),
        ),
        child: Text(label, style: TextStyle(color: active ? Colors.white : Colors.black)),
      ),
    );
  }
}
