
name: Android APK Gyarto
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Forraskod letöltese
        uses: actions/checkout@v4

      - name: Java beallitasa
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Flutter beallitasa
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable'

      - name: Projekt letrehozasa
        run: flutter create --org com.soforapp tiszta_app

      - name: Kod cseréje mukodo verziora
        run: |
          cat << 'EOF' > tiszta_app/lib/main.dart
          import 'dart:async';
          import 'package:flutter/material';
          import 'package:geolocator/geolocator.dart';

          void main() => runApp(const MaterialApp(home: DashboardScreen(), debugShowCheckedModeBanner: false));

          class DashboardScreen extends StatefulWidget {
            const DashboardScreen({super.key});
            @override
            State<DashboardScreen> createState() => _DashboardScreenState();
          }

          class _DashboardScreenState extends State<DashboardScreen> {
            String _speed = "0 km/h";
            String _timerText = "04:30:00";
            Color _statusColor = const Color(0xFF4CAF50);
            String _statusText = "VEZETHETŐ IDŐ";
            Timer? _timer;
            int _timeLeft = 16200;
            bool _running = false;
            StreamSubscription<Position>? _gpsSub;

            @override
            void initState() {
              super.initState();
              _initGPS();
            }

            Future<void> _initGPS() async {
              if (!await Geolocator.isLocationServiceEnabled()) return;
              LocationPermission perm = await Geolocator.checkPermission();
              if (perm == LocationPermission.denied) {
                perm = await Geolocator.requestPermission();
                if (perm == LocationPermission.denied) return;
              }
              _gpsSub = Geolocator.getPositionStream(
                locationSettings: const LocationSettings(accuracy: LocationAccuracy.high, distanceFilter: 1),
              ).listen((pos) {
                if (mounted) setState(() => _speed = "${(pos.speed * 3.6).toInt()} km/h");
              });
            }

            void _toggleTimer() {
              if (_running) {
                _timer?.cancel();
                setState(() => _running = false);
              } else {
                _running = true;
                _timer = Timer.periodic(const Duration(seconds: 1), (t) {
                  if (_timeLeft > 0 && mounted) {
                    setState(() {
                      _timeLeft--;
                      int h = _timeLeft ~/ 3600;
                      int m = (_timeLeft % 3600) ~/ 60;
                      int s = _timeLeft % 60;
                      
                      String sh = h.toString().padLeft(2, '0');
                      String sm = m.toString().padLeft(2, '0');
                      String ss = s.toString().padLeft(2, '0');
                      _timerText = "$sh:$sm:$ss";

                      if (_timeLeft <= 900) {
                        _statusColor = const Color(0xFFE53935);
                        _statusText = "KÖTELEZŐ PIHENŐ!";
                      } else if (_timeLeft <= 1800) {
                        _statusColor = const Color(0xFFFFB300);
                        _statusText = "MINDEGYRE PIHENŐ";
                      }
                    });
                  } else {
                    _timer?.cancel();
                  }
                });
              }
            }

            @override
            void dispose() {
              _timer?.cancel();
              _gpsSub?.cancel();
              super.dispose();
            }

            @override
            Widget build(BuildContext context) {
              return Scaffold(
                body: SafeArea(
                  child: Padding(
                    padding: const EdgeInsets.all(24.0),
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                      children: [
                        Container(
                          width: double.infinity, height: 120,
                          decoration: BoxDecoration(color: _statusColor, borderRadius: BorderRadius.circular(16)),
                          child: Center(child: Text(_statusText, style: const TextStyle(color: Colors.white, fontSize: 24, fontWeight: FontWeight.bold))),
                        ),
                        Column(children: [
                          const Text("AKTUÁLIS SEBESSÉG", style: TextStyle(color: Colors.grey)),
                          Text(_speed, style: const TextStyle(fontSize: 48, fontWeight: FontWeight.black, color: Colors.blue)),
                        ]),
                        Column(children: [
                          const Text("HÁTRALÉVŐ IDŐ", style: TextStyle(color: Colors.grey)),
                          Text(_timerText, style: const TextStyle(fontSize: 56, fontWeight: FontWeight.bold)),
                        ]),
                        ElevatedButton(
                          onPressed: _toggleTimer,
                          style: ElevatedButton.styleFrom(backgroundColor: Colors.blue, minimumSize: const Size(double.infinity, 56)),
                          child: Text(_running ? "STOP" : "INDÍTÁS", style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
                        ),
                      ],
                    ),
                  ),
                ),
              );
            }
          }
          EOF

      - name: GPS Csomag beallitasa
        run: |
          cd tiszta_app
          flutter pub add geolocator
          flutter pub get

      - name: APK Legyartasa
        run: |
          cd tiszta_app
          flutter build apk --release

      - name: Kesz APK mentese letolthetokent
        uses: actions/upload-artifact@v4
        with:
          name: sofor-applikacio
          path: tiszta_app/build/app/outputs/flutter-apk/app-release.apk
