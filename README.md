import 'dart:io';
import 'package:flutter/material.dart';
import 'package:file_picker/file_picker.dart';
import 'package:excel/excel.dart';
import 'package:path_provider/path_provider.dart';
import 'package:open_file/open_file.dart';

void main() {
  runApp(const SdrToExcelApp());
}

class SdrToExcelApp extends StatelessWidget {
  const SdrToExcelApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'SDR to Excel',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: const HomePage(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class HomePage extends StatefulWidget {
  const HomePage({super.key});
  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  File? sdrFile;
  String? excelPath;
  bool isConverting = false;
  String status = '';

  Future<void> pickFile() async {
    FilePickerResult? result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: ['txt'],
    );
    if (result != null && result.files.single.path != null) {
      setState(() {
        sdrFile = File(result.files.single.path!);
        excelPath = null;
        status = '';
      });
    }
  }

  Future<void> convertToExcel() async {
    if (sdrFile == null) return;
    setState(() {
      isConverting = true;
      status = 'جاري التحويل...';
    });

    try {
      List<List<dynamic>> rows = [];
      final lines = await sdrFile!.readAsLines(encoding: utf8);
      // Add Excel header
      rows.add(['Point_ID', 'Easting', 'Northing', 'Elevation', 'Code']);
      for (var line in lines) {
        final parsed = parseSdrLine(line);
        if (parsed != null) rows.add(parsed);
      }

      // Create Excel
      var excel = Excel.createExcel();
      Sheet sheet = excel['Sheet1'];
      for (var row in rows) {
        sheet.appendRow(row);
      }

      // Save file
      final dir = await getApplicationDocumentsDirectory();
      String outPath = '${dir.path}/sdr_output_${DateTime.now().millisecondsSinceEpoch}.xlsx';
      final fileBytes = excel.encode();
      if (fileBytes != null) {
        File(outPath)
          ..createSync(recursive: true)
          ..writeAsBytesSync(fileBytes);
        setState(() {
          excelPath = outPath;
          status = 'تم إنشاء ملف Excel:\n$outPath';
        });
      } else {
        setState(() {
          status = 'حدث خطأ أثناء حفظ الملف.';
        });
      }
    } catch (e) {
      setState(() {
        status = 'خطأ أثناء التحويل: $e';
      });
    } finally {
      setState(() {
        isConverting = false;
      });
    }
  }

  // Parsing logic (Dart version)
  List<dynamic>? parseSdrLine(String line) {
    final tokens = line.trim().split(RegExp(r'[\s,;]+'));
    if (tokens.isEmpty || (tokens.length == 1 && tokens[0] == '')) return null;
    final numericTokens = tokens.where((t) => isNumber(t)).toList();
    if (numericTokens.length < 4) return null;

    final pointId = numericTokens[0];
    final easting = numericTokens[1];
    final northing = numericTokens[2];
    final elevation = numericTokens[3];
    final nonNumeric = tokens.where((t) => !isNumber(t)).toList();
    final code = nonNumeric.isNotEmpty ? nonNumeric.last : '';
    return [pointId, easting, northing, elevation, code];
  }

  bool isNumber(String s) {
    return RegExp(r'^[-+]?\d*\.?\d+(?:[eE][-+]?\d+)?$').hasMatch(s);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('SDR ➔ Excel بدون إنترنت')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            ElevatedButton.icon(
              onPressed: isConverting ? null : pickFile,
              icon: const Icon(Icons.attach_file),
              label: const Text('اختر ملف SDR (.txt)'),
            ),
            const SizedBox(height: 10),
            if (sdrFile != null)
              Text('الملف: ${sdrFile!.path}', style: const TextStyle(fontSize: 13)),
            const SizedBox(height: 20),
            ElevatedButton.icon(
              onPressed: (sdrFile != null && !isConverting) ? convertToExcel : null,
              icon: const Icon(Icons.file_download),
              label: const Text('حوّل إلى Excel'),
            ),
            const SizedBox(height: 20),
            if (isConverting) const CircularProgressIndicator(),
            if (status.isNotEmpty)
              Padding(
                padding: const EdgeInsets.all(8.0),
                child: Text(status, style: const TextStyle(fontSize: 14, color: Colors.green)),
              ),
            const Spacer(),
            if (excelPath != null)
              ElevatedButton.icon(
                icon: const Icon(Icons.open_in_new),
                label: const Text('فتح ملف Excel'),
                onPressed: () async {
                  if (excelPath != null) {
                    await OpenFile.open(excelPath!);
                  }
                },
              ),
          ],
        ),
      ),
    );
  }
}
