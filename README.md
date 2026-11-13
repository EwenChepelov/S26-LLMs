# S26-LLMs — Galaxy S26 (Exynos 2600 / E9965) × Google AI Edge Gallery

Livrables de la session (voir `REPORT.md` pour le rapport complet) :

- `REPORT.md` — rapport détaillé par étape (fait / bloqué / non vérifié, preuves).
- `gallery-exynos2600-changes.diff` — diff minimal appliqué sur `google-ai-edge/gallery` (branche `vibe/exynos2600-s26-86b402`, commit `e579ee1`) : `isExynos2600()`, CPU par défaut + option NPU, repli GPU→CPU (ticket LiteRT-LM #2114), `applicationId=com.example.gallery.exynos`, déclaration `libenn_public_api_cpp.so`.
- `libLiteRtDispatch_Samsung.so` — construit depuis `google-ai-edge/LiteRT` :
  `bazel build --config=android_arm64 //litert/vendors/samsung/dispatch:dispatch_api_so`
  (AArch64, SONAME vérifié ; à déposer dans `Android/src/app/src/main/jniLibs/arm64-v8a/`).
- `app-debug.apk` (353 Mo) — non inclus dans git (limite GitHub 100 Mo) ; rebuild local :
  ```
  git apply gallery-exynos2600-changes.diff   # dans google-ai-edge/gallery
  cd Android/src && ./gradlew assembleDebug
  ```
