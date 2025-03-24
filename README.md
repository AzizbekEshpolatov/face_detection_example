# Face Detection Example  
Bu loyiha Flutter va Google ML Kit yordamida yuzni aniqlash uchun ishlatiladi. Ilova kameradan foydalangan holda foydalanuvchi yuzining turli holatlarini aniqlaydi va ko‘rsatmalar beradi.  
---

## Dependencies
Loyiha quyidagi paketlardan foydalanadi:  
```yaml
dependencies:
  flutter:
    sdk: flutter
  google_mlkit_face_detection: ^0.9.1
  camera: ^0.10.5+9
  permission_handler: ^11.3.0
```
## 🌍 Tilga Mos Tarjimalar (Translations)  
Loyiha uch tilda qo'llab-quvvatlaydi: **Uzbek, English, Russian**  

## Code
```dart
import 'dart:math';
import 'package:car_land/core/common/words.dart';
import 'package:car_land/core/extension/message_extension.dart';
import 'package:car_land/core/extension/size_extension.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:camera/camera.dart';
import 'package:google_mlkit_face_detection/google_mlkit_face_detection.dart';
import 'package:permission_handler/permission_handler.dart';

class FaceDetectionPage extends StatefulWidget {
  const FaceDetectionPage({super.key});

  @override
  State<FaceDetectionPage> createState() => _FaceDetectionPageState();
}

class _FaceDetectionPageState extends State<FaceDetectionPage> {
  CameraController? _cameraController;
  FaceDetector? _faceDetector;
  bool _isDetecting = false;
  List<Face> _faces = [];
  String _currentTask = 'take_action';
  String _displayMessage = Words.faceNotDetected.tr();
  int _countdown = 0;
  bool _isCountingDown = false;
  final List<String> _tasks = [
    "look_left",
    "look_right",
    "look_up",
    "look_down",
    "smile",
    "close_eyes"
  ];
  final Random _random = Random();
  bool _isCameraInitialized = false;

  @override
  void initState() {
    super.initState();
    _initializeFaceDetector();
    _requestCameraPermission();
  }

  void _initializeFaceDetector() {
    final options = FaceDetectorOptions(
      enableContours: true,
      enableLandmarks: true,
      enableClassification: true,
      performanceMode: FaceDetectorMode.fast,
    );
    _faceDetector = FaceDetector(options: options);
  }

  Future<void> _requestCameraPermission() async {
    var status = await Permission.camera.status;
    if (!status.isGranted) status = await Permission.camera.request();

    if (status.isGranted) {
      _initializeCamera();
    } else if (status.isDenied) {
      showErrors(Words.cameraPermissionDenied.tr());
    } else if (status.isPermanentlyDenied) {
      showErrors(Words.cameraPermissionPermanentlyDenied.tr());
      await openAppSettings();
    }
  }

  Future<void> _initializeCamera() async {
    try {
      final cameras = await availableCameras();
      if (cameras.isEmpty) {
        showErrors(Words.noCameraFound.tr());
        return;
      }

      final frontCamera = cameras.firstWhere(
        (camera) => camera.lensDirection == CameraLensDirection.front,
        orElse: () => cameras.first,
      );

      _cameraController = CameraController(
        frontCamera,
        ResolutionPreset.high,
        enableAudio: false,
        imageFormatGroup: defaultTargetPlatform == TargetPlatform.android
            ? ImageFormatGroup.nv21
            : ImageFormatGroup.jpeg,
      );

      await _cameraController!.initialize();
      if (mounted) {
        setState(() => _isCameraInitialized = true);
      }
      _cameraController?.startImageStream(_processCameraImage);
    } catch (e) {
      showErrors("${Words.cameraInitError.tr()}:${e.toString()}");
    }
  }

  Future<void> _processCameraImage(CameraImage image) async {
    if (_isDetecting || _isCountingDown) return;
    _isDetecting = true;
    try {
      final WriteBuffer allBytes = WriteBuffer();
      for (final Plane plane in image.planes) {
        allBytes.putUint8List(plane.bytes);
      }
      final Uint8List bytes = allBytes.done().buffer.asUint8List();

      final Size imageSize = Size(
        image.width.toDouble(),
        image.height.toDouble(),
      );

      final InputImageRotation imageRotation = _getImageRotation();

      final bytesPerRow = image.planes[0].bytesPerRow;
      if (bytesPerRow == 0) {
        debugPrint("Xatolik: bytesPerRow 0 ga teng. Buni tuzatish lozim.");
        _isDetecting = false;
        return;
      }

      final inputImageData = InputImageMetadata(
        size: imageSize,
        rotation: imageRotation,
        format: defaultTargetPlatform == TargetPlatform.android
            ? InputImageFormat.nv21
            : InputImageFormat.bgra8888,
        bytesPerRow: bytesPerRow,
      );

      final inputImage = InputImage.fromBytes(
        bytes: bytes,
        metadata: inputImageData,
      );

      final List<Face> faces = await _faceDetector!.processImage(inputImage);

      if (faces.isNotEmpty) {
        _checkTaskCompletion(faces.first, inputImage.filePath);
        setState(() {
          _faces = faces;
          if (_currentTask != 'hold_still') {
            _displayMessage = _currentTask == 'take_action'
                ? Words.takeAction.tr()
                : _currentTask == 'done'
                    ? Words.done.tr()
                    : _currentTask == 'look_left'
                        ? Words.lookLeft.tr()
                        : _currentTask == 'look_right'
                            ? Words.lookRight.tr()
                            : _currentTask == 'look_up'
                                ? Words.lookUp.tr()
                                : _currentTask == 'look_down'
                                    ? Words.lookDown.tr()
                                    : _currentTask == 'smile'
                                        ? Words.smile.tr()
                                        : Words.closeEyes.tr();
          }
        });
      } else {
        setState(() {
          _faces = [];
          _displayMessage = Words.faceNotDetected.tr();
        });
      }
      debugPrint('Aniqlangan yuzlar soni: ${faces.length}');
    } catch (e) {
      debugPrint("Rasmni qayta ishlashda xato: $e");
      setState(() => _displayMessage = 'Xato: $e');
    } finally {
      _isDetecting = false;
    }
  }

  void _checkTaskCompletion(Face face, String? path) {
    bool taskCompleted = false;
    switch (_currentTask) {
      case "look_right":
        if (face.headEulerAngleY != null && face.headEulerAngleY! < -15) {
          _tasks.remove("look_right");
          taskCompleted = true;
        }
        break;
      case "look_left":
        if (face.headEulerAngleY != null && face.headEulerAngleY! > 15) {
          _tasks.remove("look_left");
          taskCompleted = true;
        }
        break;
      case "look_up":
        if (face.headEulerAngleX != null && face.headEulerAngleX! > 15) {
          _tasks.remove("look_up");
          taskCompleted = true;
        }
        break;
      case "look_down":
        if (face.headEulerAngleX != null && face.headEulerAngleX! < -15) {
          _tasks.remove("look_down");
          taskCompleted = true;
        }
        break;
      case "smile":
        if (face.smilingProbability != null && face.smilingProbability! > 0.7) {
          _tasks.remove("smile");
          taskCompleted = true;
          _startCountdown();
        }
        break;
      case "close_eyes":
        if ((face.leftEyeOpenProbability != null &&
                face.leftEyeOpenProbability! < 0.3) &&
            (face.rightEyeOpenProbability != null &&
                face.rightEyeOpenProbability! < 0.3)) {
          _tasks.remove("close_eyes");
          taskCompleted = true;
        }
        break;
    }

    if (taskCompleted || _currentTask == 'take_action') {
      if (_currentTask != 'smile') {
        setState(() {
          _currentTask = _tasks.isNotEmpty
              ? _tasks[_random.nextInt(_tasks.length)]
              : 'done';
        });
      }
    }

    if (_tasks.isEmpty && !_isCountingDown) _startCountdown();
  }

  void _startCountdown() {
    setState(() {
      _currentTask = 'hold_still';
      _isCountingDown = true;
      _countdown = 0;
      _displayMessage = Words.holdStillCountdown.tr();
    });
    Future.doWhile(() async {
      await Future.delayed(const Duration(seconds: 1));
      setState(() {
        _countdown++;
        _displayMessage = '${Words.holdStillCountdown.tr()} $_countdown';
      });

      if (_countdown == 3) {
        _takePictureAndSendToBackend();
        return false;
      }
      return true;
    });
  }

  Future<void> _takePictureAndSendToBackend() async {
    try {
      final XFile? picture = await _cameraController?.takePicture();
      if (picture != null) {
        debugPrint("Backendga yuborildi: ${picture.path}");
        showSuccess("Success action!");
      } else {
        debugPrint("Rasm olishda xato: Rasm topilmadi");
        showErrors('Rasm olishda xato: Rasm topilmadi');
      }
    } catch (e) {
      debugPrint("Rasm olishda xato: $e");
      showErrors('Rasm olishda xato: $e');
    } finally {
      setState(() {
        _isCountingDown = false;
        _currentTask = 'done';
        _displayMessage = Words.done.tr();
      });
    }
  }

  InputImageRotation _getImageRotation() {
    final int sensorOrientation =
        _cameraController!.description.sensorOrientation;
    switch (sensorOrientation) {
      case 90:
        return InputImageRotation.rotation90deg;
      case 180:
        return InputImageRotation.rotation180deg;
      case 270:
        return InputImageRotation.rotation270deg;
      default:
        return InputImageRotation.rotation0deg;
    }
  }

  @override
  void dispose() {
    _cameraController?.dispose();
    _faceDetector?.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (!_isCameraInitialized ||
        _cameraController == null ||
        !_cameraController!.value.isInitialized) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    final size = MediaQuery.of(context).size;
    final cameraPreviewSize = Size(
      _cameraController!.value.previewSize!.height,
      _cameraController!.value.previewSize!.width,
    );

    return Scaffold(
      body: Stack(
        fit: StackFit.expand,
        children: [
          CameraPreview(_cameraController!),
          Positioned(
            top: 50,
            left: 0,
            right: 0,
            child: Center(
              child: Container(
                padding: const EdgeInsets.all(10),
                decoration: BoxDecoration(
                  color: Colors.black54,
                  borderRadius: BorderRadius.circular(10),
                ),
                child: Text(
                  _displayMessage,
                  style: const TextStyle(color: Colors.white, fontSize: 20),
                ),
              ),
            ),
          ),
          ..._faces.map((face) {
            final rect = face.boundingBox;

            final scaleX = size.width / cameraPreviewSize.width;
            final scaleY = size.height / cameraPreviewSize.height;

            return Positioned(
              left: rect.left * scaleX,
              top: rect.top * scaleY,
              width: rect.width * scaleX,
              height: rect.height * scaleY,
              child: Container(
                decoration: BoxDecoration(
                  border: Border.all(color: Colors.red, width: 2.w),
                ),
              ),
            );
          }),
        ],
      ),
    );
  }
}

```

### tarjimalar  
```dart
final Map<String, String> taskMessagesUz = {
  'takeAction': "Harakatni boshlash",
  'done': "Tugallandi",
  'lookLeft': "Chapga qarang",
  'lookRight': "O‘ngga qarang",
  'lookUp': "Tepaga qarang",
  'lookDown': "Pastga qarang",
  'smile': "Tabassum qiling",
  'closeEyes': "Ko‘zingizni yuming",
  'holdStill': "Qimirlamang",
  'faceNotDetected': "Yuz aniqlanmadi",
  'cameraPermissionPermanentlyDenied': 
      "Kamera ruxsati doimiy rad etilgan. Iltimos, sozlamalardan ruxsat bering.",
  'cameraPermissionDenied': "Kamera ruxsati berilmadi! Iltimos, ruxsat bering.",
  'noCameraFound': "Hech qanday kamera topilmadi!",
  'cameraInitError': "Kamera initsializatsiyasida xato:",
  'holdStillCountdown': "Qimirlamay turing: ",
};
final Map<String, String> taskMessagesRu = {
  'takeAction': "Начните действие",
  'done': "Завершено",
  'lookLeft': "Посмотрите налево",
  'lookRight': "Посмотрите направо",
  'lookUp': "Посмотрите вверх",
  'lookDown': "Посмотрите вниз",
  'smile': "Улыбайтесь",
  'closeEyes': "Закройте глаза",
  'holdStill': "Замрите",
  'faceNotDetected': "Лицо не обнаружено",
  'cameraPermissionPermanentlyDenied': 
      "Доступ к камере был отклонен навсегда. Разрешите доступ в настройках.",
  'cameraPermissionDenied': "Доступ к камере отклонен! Пожалуйста, разрешите доступ.",
  'noCameraFound': "Камера не найдена!",
  'cameraInitError': "Ошибка инициализации камеры:",
  'holdStillCountdown': "Замрите: ",
};
final Map<String, String> taskMessagesEn = {
  'takeAction': "Start the action",
  'done': "Completed",
  'lookLeft': "Look to the left",
  'lookRight': "Look to the right",
  'lookUp': "Look up",
  'lookDown': "Look down",
  'smile': "Smile",
  'closeEyes': "Close your eyes",
  'holdStill': "Hold still",
  'faceNotDetected': "Face not detected",
  'cameraPermissionPermanentlyDenied': 
      "Camera permission is permanently denied. Please allow it from settings.",
  'cameraPermissionDenied': "Camera permission denied! Please allow access.",
  'noCameraFound': "No camera found!",
  'cameraInitError': "Camera initialization error:",
  'holdStillCountdown': "Hold still: ",
};
```
