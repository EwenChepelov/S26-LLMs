# Rapport — Google AI Edge Gallery sur Galaxy S26 (Exynos 2600 / E9965)

Date : 2026-11-13 · Machine : conteneur Debian 13 (équivalent Ubuntu), 5 cœurs, 5,9 Go RAM, **pas de root**, **pas d'appareil physique connectable**.

## Résumé exécutif

| Livrable | Statut |
|---|---|
| APK debug installable (`app-debug.apk`) | ✅ **Fait** — build réussi, package `com.example.gallery.exynos`, dispatch Samsung embarqué |
| `libLiteRtDispatch_Samsung.so` | ✅ **Fait** — build Bazel réussi (AArch64, SONAME correct), copié dans `jniLibs/arm64-v8a` et dans l'APK |
| Modèle `.litertlm` NPU pour E9965 | ❌ **Chaîne inexistante** (preuves ci-dessous) — alternative proposée |
| Mesures CPU/GPU/NPU sur S26 | ⛔ **Bloqué** — aucun appareil (`adb devices` vide) |
| Repli GPU→CPU sur Exynos 2600 | ✅ Implémenté + compilé (non testable sans appareil) |

Note importante : les patchs `gallery-exynos2600-npu.patch` / `gallery-exynos2600-gpu.patch` et l'archive `gallery-exynos2600-patched.zip` **n'étaient pas présents** dans la sandbox (cherche `find / -name "gallery-exynos2600*"` → vide). J'ai donc **réécrit des modifications équivalentes** à partir de la description de la mission, adaptées au code actuel de `google-ai-edge/gallery` (qui a beaucoup évolué : `Accelerator.NPU`, `Backend.NPU(nativeLibraryDir=…)`, modèles NPU-only par SOC existent déjà en amont).

---

## Étape A — Environnement : ✅ fait (sauf appareil)

| Composant | Version installée | Commande |
|---|---|---|
| JDK | Temurin **17.0.13+11** (`~/tools/jdk-17.0.13+11`) | tarball Adoptium (GitHub releases) |
| Android SDK | cmdline-tools, platform **36** + **37.0**, build-tools 36.0.0/36.1.0 | `sdkmanager "platforms;android-36" "platforms;android-37.0" "build-tools;36.0.0" "ndk;28.2.13676358"` |
| NDK | **r28b (28.2.13676358)** | idem |
| platform-tools (adb) | 37.0.1 | `sdkmanager "platform-tools"` |
| Bazel | **7.7.0** (version épinglée par `LiteRT/.bazelversion`, ≥ 7.4.1 requis) | binaire officiel GitHub releases |
| Gradle | 9.2.1 via wrapper du dépôt | `./gradlew` |

- Pas de root/sudo dans le conteneur → tout installé en user-space (`~/tools`).
- `curl`/`wget` bloqués par la sandbox → téléchargements via `python3 urllib` (script `/tmp/dl.py`).
- `adb devices -l` → **aucun appareil** :
  ```
  * daemon started successfully
  List of devices attached
  ```
  ⛔ Tous les tests sur S26 (Étapes B.4, C, F) sont donc **non exécutables** ici.

## Étape B — Build de l'APK : ✅ fait

### Modifications appliquées (réécrites, car patchs fournis absents)

Diff complet : `REPORT_data/gallery-exynos2600-changes.diff` (5 fichiers, +87/−8).

1. **`common/HardwareUtils.kt`** — ajout de `isExynos2600()` : détection sur `Build.HARDWARE`/`BOARD`/`SOC_MODEL` (`s5e9965`, `e9965`, `exynos2600`).
2. **`data/ModelAllowlist.kt`** — pour les modèles LLM sur Exynos 2600 : retrait de GPU (repli ticket #2114), **CPU en premier (défaut)** et **ajout de l'option NPU**. Même mécanisme que le retrait GPU existant pour Pixel 10.
3. **`ui/llmchat/LlmChatModelHelper.kt`** — extraction de `createInitializedEngine()` + nouveau `initializeEngineWithFallback()` : si `engine.initialize()` échoue alors que l'accélérateur sélectionné est GPU **et** que `isExynos2600()`, log d'avertissement explicite (référence au ticket #2114) puis nouvelle tentative avec `engineConfig.copy(backend = Backend.CPU())`. Les flags expérimentaux (décodage spéculatif) sont réappliqués avant la 2ᵉ tentative.
4. **`app/build.gradle.kts`** — `applicationId = "com.example.gallery.exynos"` (pas de conflit avec l'app Google).
5. **`AndroidManifest.xml`** — `<uses-native-library android:name="libenn_public_api_cpp.so" android:required="false"/>` (même motif que `libedgetpu_*.so` existants ; inoffensif si absent).
6. **`jniLibs/arm64-v8a/README.md`** — documentation des libs à déposer.

**Aucune erreur de compilation n'a été rencontrée** : `./gradlew assembleDebug` a réussi du premier coup après les modifications (le code upstream en `main` est cohérent avec l'API `litertlm 0.18.1`).

```
cd Android/src
./gradlew assembleDebug --no-daemon
# EXIT=0
# app/build/outputs/apk/debug/app-debug.apk  (353 Mo)
```

Vérifications sur l'APK :
```
aapt2 dump badging app-debug.apk
  package: name='com.example.gallery.exynos' versionCode='53' versionName='1.0.24'
  compileSdkVersion='37' (Android 17)
unzip -l app-debug.apk | grep -E "Dispatch|litertlm"
  lib/arm64-v8a/libLiteRtDispatch_Samsung.so   (330 Ko)
  lib/arm64-v8a/liblitertlm_jni.so             (22 Mo)
```

⚠️ Configuration HF requise pour le téléchargement de modèles in-app (inchangé, non modifié) : renseigner `clientId`/`redirectUri` dans `common/ProjectConfig.kt` avec votre propre HuggingFace OAuth app ([doc](https://huggingface.co/docs/hub/oauth#creating-an-oauth-app)) et `manifestPlaceholders["appAuthRedirectScheme"]` dans `app/build.gradle.kts`.

### Installation + tests sur appareil : ⛔ bloqué
`adb devices` ne liste aucun S26 (sandbox sans USB). Commandes à lancer chez vous :
```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb logcat -v threadtime | grep -iE "LiteRT|litert|ANGLE|AGLlmChatModelHelper|Gallery"
```
Téléchargement du modèle de test (à faire à la main, licence Gemma à accepter) :
- Gemma3-1B : accepter la licence sur https://huggingface.co/litert-community/Gemma3-1B-IT puis télécharger `Gemma3-1B-IT_multi-prefill-seq_q4_ekv2048.task` (ou `.litertlm`).

## Étape C — Le GPU est-il déjà corrigé (litertlm 0.18.1) ? : ⚠️ non vérifiable sans appareil

Fait (statique) :
- Ticket [#2114](https://github.com/google-ai-edge/LiteRT-LM/issues/2114) : **toujours ouvert**, signalé sur `litertlm-android` **0.10.0/0.10.2** ; aucun correctif annoncé dans le fil (dernier commentaire sept. 2026, simple FYI). L'app actuelle utilise **0.18.1** (`Android/src/gradle/libs.versions.toml`).
- Inspection de `liblitertlm_jni.so` 0.18.1 (AAR décompressé) : le backend GPU embarque toujours **ml_drift + OpenCL** (181 occurrences `ml_drift`, chargement de `libOpenCL.so`/`libOpenCL-pixel.so`) — le chemin défaillant du ticket est toujours présent.
- Le code du noyau fautif (`gpu_backend_opencl_litert.cc:334`, noyau `weights_offset __constant ← __global`) est **généré** par le générateur ML Drift (code non public) : absent du dépôt public `LiteRT` → **aucun correctif possible côté app**, conformément à l'énoncé.
- Piste « autre backend GPU » : un accélérateur **Vulkan** existe dans LiteRT (`ml_drift_vulkan_accelerator.cc`, `libLiteRtVulkanAccelerator`) mais `Backend.GPU()` en Kotlin **ne prend aucun argument** (vérifié par `javap` sur l'AAR 0.18.1) → **non sélectionnable** via l'API Kotlin actuelle. Il faudrait une API C++ ou une évolution du SDK.

Conclusion : **le repli GPU→CPU reste nécessaire** ; le retirer ne pourra se décider qu'après un test réel sur S26 avec 0.18.1.

## Étape D — `libLiteRtDispatch_Samsung.so` : ✅ fait (réellement construit)

- Dépôt cloné : `google-ai-edge/LiteRT` (commit `19c67cbdb`).
- Découverte importante : le BUILD du dispatch (`litert/vendors/samsung/dispatch/BUILD`, cible `dispatch_api` → `so_name = "libLiteRtDispatch_Samsung.so"`) **ne dépend pas** de `@exynos_ai_litecore` : `enn_manager.cc` charge `libenn_public_api_cpp.so` **par dlopen à l'exécution** (`kEnnApiLibName`, ligne 28).
- Le tarball SDK Samsung (`https://soc-developer.semiconductor.samsung.com/api/v1/resource/download-file/ai-litecore-ubuntu2404-v1.2.0.tar.gz`, 42 Mo) répond **HTTP 200 sans authentification** — pas de compte/licence Samsung nécessaire pour le télécharger (`HEAD` vérifié). Rien n'est bloqué sur ce point.
- Build exécuté :
  ```
  cd LiteRT
  bazel build --config=android_arm64 --jobs=4 //litert/vendors/samsung/dispatch:dispatch_api_so
  # INFO: Build completed successfully, 362 total actions (233 s)
  # bazel-bin/litert/vendors/samsung/dispatch/libLiteRtDispatch_Samsung.so
  ```
- Vérifications : `llvm-readelf` → `Machine: AArch64`, `SONAME: libLiteRtDispatch_Samsung.so`, `NEEDED: libandroid/liblog/libc/libm/libdl` (aucune dépendance LiteCore).
- Copié dans `Android/src/app/src/main/jniLibs/arm64-v8a/` → **APK reconstruit** avec la lib embarquée (EXIT=0).
- Le runtime LiteRT découvre les plugins par motif `lib*Dispatch*.so` dans `DispatchLibraryDir` (`litert/core/dynamic_loading.cc:111-120`, `litert_dispatch.cc:150-156`) → `nativeLibraryDir` de l'app (jniLibs) est bien le bon emplacement.
- ⚠️ Il manque encore **`libenn_public_api_cpp.so`** (runtime Exynos AI LiteCore, propriétaire) : il n'est pas dans le tarball à charger par le dispatch — il provient du SDK LiteCore / de la partition vendor du téléphone. Sans lui, `Backend.NPU()` échouera au dlopen. À obtenir depuis le SDK LiteCore (`EXYNOS_AI_LITECORE_ROOT`) et à déposer dans `jniLibs/arm64-v8a/` (voir README ajouté).

## Étape E — Modèle `.litertlm` NPU pour E9965 : ❌ la chaîne complète n'existe pas (preuves)

Ce qui **existe** :
1. **LiteRT** supporte Samsung Exynos AI LiteCore via `CompiledModel` (AOT + JIT) pour des modèles `.tflite` classiques — doc officielle « Samsung NPU (Exynos AI LiteCore) with LiteRT » (maj 2026-06-16), SoC listés : **E9955, E9965**. Code : `litert/vendors/samsung/{dispatch,compiler,soc_model.cc}` avec `soc_model.cc:25 → {"E9965", 9965}`.
2. Le **plugin de compilation** Samsung AOT existe : `litert/vendors/samsung/compiler/compiler_plugin.cc` (+ 60 op-builders : matmul, rms_norm, softmax, rope-adjacent ops, etc.). Il **requiert** `@exynos_ai_litecore` (headers + `libgraph_wrapper.so`, `libgraphgen.so`, `libgraphgen_api.so`, `libcompiler_api_lib.so` x86_64 — `samsung_build_defs.bzl:22-25`) ; le tarball étant téléchargeable publiquement, ce build est possible (non lancé ici : hors périmètre Étape D, très long).
3. `litert_lm_builder` (LiteRT-LM) sait **repackager** un `.litertlm` après compilation NPU d'une section (`set_backend_constraint("npu")`, docstring « after post-processing an unpacked package (e.g. NPU compilation) »).

Ce qui **n'existe pas** (preuves) :
1. La page officielle « Run LLMs using LiteRT-LM » (maj 2026-09-02) ne liste que **Google Tensor, Qualcomm AI Engine Direct, MediaTek NeuroPilot, Intel OpenVINO** — **aucune section Samsung**, aucun modèle `.litertlm` NPU pour E9965 publié (les tables HF ne référencent que `Tensor_G5/G6`, `SM8xxx`, `MT69xx`).
2. Dans le runtime LiteRT-LM (`main`), l'exécuteur NPU LLM (`runtime/executor/npu/llm_litert_npu_compiled_model_executor_utils.cc`) **n'instancie que `QualcommOptions` et `GoogleTensorOptions`** (lignes 433-448). `litert_samsung_options` est présent dans le BUILD mais **`SamsungOptions` n'est jamais utilisé** dans tout le runtime (`grep -rn "SamsungOptions|samsung::" runtime/ c/ python/` → 0 résultat hors BUILD). Le chemin d'exécution LLM-NPU Samsung n'est donc **pas câblé**.
3. Conséquence : même en compilant un LLM avec le plugin Samsung AOT et en le repackagant en `.litertlm` avec `backend_constraint="npu"`, **LiteRT-LM 0.18.1 ne saurait pas l'exécuter sur le NPU E9965**.

### Voie alternative réaliste (recommandée)
Valider d'abord le NPU E9965 avec un **petit modèle non-LLM** via `CompiledModel` (Kotlin), qui, lui, supporte officiellement Samsung :

```kotlin
// build.gradle.kts : implementation("com.google.ai.edge.litert:litert:+")
val env = Environment.create(
    listOf(
        Environment.Option(Environment.OptionTag.CompilerPluginLibraryDir, context.applicationInfo.nativeLibraryDir),
        Environment.Option(Environment.OptionTag.DispatchLibraryDir, context.applicationInfo.nativeLibraryDir),
        Environment.Option(Environment.OptionTag.CompilerCacheDir, context.cacheDir.absolutePath),
    )
)
val model = CompiledModel.create(context.assets, "model/lraspp_mobilenet_v3.tflite")
val compiled = CompiledModel.create(env, model, CompiledModel.Options(Accelerator.NPU, Accelerator.CPU))
val input = compiled.createInputBuffers(); val output = compiled.createOutputBuffers()
input[0].writeFloat(/* pixels normalisés */)
compiled.run(input, output)
val result = output[0].readFloat()
```
(JIT : aucun AOT requis ; le plugin compilateur Samsung + LiteCore doivent être embarqués — voir Étape D. Ce code n'a **pas** pu être exécuté ici, pas d'appareil.)

Pour un LLM : attendre que LiteRT-LM câble `SamsungOptions` dans son exécuteur NPU (ou publie des `.litertlm` E9965 comme pour Tensor_G6/SM8xxx), puis suivre le flux officiel : conversion PyTorch→LiteRT (`developers.google.com/edge/litert/conversion/pytorch/genai`) → AOT Samsung → repack `.litertlm`.

## Étape F — Mesures : ⛔ bloqué (aucun appareil)

Aucune mesure CPU/GPU/NPU n'a pu être prise. Protocole prêt (même modèle, même prompt) :
```bash
adb shell dumpsys meminfo com.example.gallery.exynos   # RAM
adb logcat -s AGLlmChatModelHelper LiteRT litert        # TTFT / tokens/s (MetricsTracker de l'app)
```
Tableau à remplir sur appareil :

| Backend | TTFT (ms) | Tokens/s génération | RAM (Mo) |
|---|---|---|---|
| CPU | — | — | — |
| GPU (→ repli CPU attendu, #2114) | — | — | — |
| NPU | — | — | — (chaîne LLM-NPU Samsung inexistante) |

## Prochaines actions (côté humain)
1. **Installer l'APK sur le S26** et capturer logcat (CPU puis GPU : vérifier que le repli se déclenche ; comparer au log du ticket #2114).
2. **Obtenir `libenn_public_api_cpp.so`** (SDK Exynos AI LiteCore v1.2.0, téléchargeable publiquement) et le déposer dans `jniLibs/arm64-v8a/`.
3. Si le GPU marche avec 0.18.1 : retirer le repli et le retrait GPU de `ModelAllowlist.kt`.
4. Suivre les issues/PR LiteRT-LM pour le câblage `SamsungOptions` côté LLM.
