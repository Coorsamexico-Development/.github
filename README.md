# Workflows reutilizables de Coorsamexico-Development

Este repo contiene workflows de GitHub Actions que las apps de la organización pueden reutilizar.

## Android Play release

Workflow: `.github/workflows/android-play-release.yml`

Construye un AAB firmado de una app Flutter y lo sube a Google Play Console. El service account de Play es uno solo para la organización; cada app aporta su package, su upload key y sus datos de build.

### Cuenta de Play

La cuenta de Google Play Console y el service account son de la organización, no por app. Para publicar una app nueva:

1. En Play Console → **Usuarios y permisos**, invitar el correo del service account.
2. Darle permiso **Release manager** (o Admin) sobre la app nueva, o a nivel de cuenta si debe publicar todas.
3. Guardar el JSON del service account como **secret de organización** `PLAY_SERVICE_ACCOUNT_JSON` con visibilidad para los repos que lo usan:

```bash
gh secret set PLAY_SERVICE_ACCOUNT_JSON \
  --org Coorsamexico-Development \
  --visibility all \
  < /ruta/al/play-service-account.json
```

Cada app conserva sus propios secrets de Android (upload key distinta):

- `ANDROID_KEYSTORE_BASE64`
- `ANDROID_STORE_PASSWORD`
- `ANDROID_KEY_PASSWORD`
- `ANDROID_KEY_ALIAS`
- `GOOGLE_SERVICES_JSON_BASE64` (opcional, si la app usa Firebase)

### Uso en una app Flutter

El repo de la app solo necesita un caller:

```yaml
name: App móvil - publicar en Play Store
on:
  push:
    tags: ['app-v*', 'app-interna-v*']
jobs:
  publicar:
    uses: Coorsamexico-Development/.github/.github/workflows/android-play-release.yml@main
    with:
      package-name: com.coorsamexicodevelopment.mobile_wms_cryd
      working-directory: .
      build-args: >-
        --release --obfuscate --split-debug-info=build/symbols
        --dart-define=API_URL=${{ vars.WMS_API_URL }}
      whatsnew-directory: store/whatsnew
    secrets: inherit
```

Tags:

- `app-interna-v1.0.0+1` → track `internal`
- `app-v1.0.0+1` → track `production`

El tag debe coincidir con `version` de `pubspec.yaml`.

### Inputs

| Input | Default | Uso |
| --- | --- | --- |
| `package-name` | — | `applicationId` de la app |
| `working-directory` | `.` | Directorio de la app dentro del repo |
| `build-args` | `--release` | Argumentos extra de `flutter build appbundle` |
| `tag-prefix` | `app` | Prefijo de tags de release |
| `keystore-path` | `android/upload-keystore.jks` | Ruta del keystore desde la raíz del repo |
| `key-properties-path` | `android/key.properties` | Ruta de `key.properties` desde la raíz |
| `store-file-name` | `upload-keystore.jks` | `storeFile` dentro de `key.properties` |
| `whatsnew-directory` | `store/whatsnew` | Notas de release |
| `symbols-path` | `build/symbols` | Símbolos de ofuscación |
| `artifact-name` | `android-aab` | Nombre del artifact |
| `java-version` | `17` | Java |
| `flutter-channel` | `stable` | Canal de Flutter |

### Ejemplo ERP

La app `mobile/hoja-recorrido-app` del repo `plataforma-coorsa-erp` puede usar:

```yaml
with:
  package-name: com.coorsa.hoja_recorrido_app
  working-directory: mobile/hoja-recorrido-app
  keystore-path: mobile/hoja-recorrido-app/android/app/upload-keystore.jks
  key-properties-path: mobile/hoja-recorrido-app/android/key.properties
  store-file-name: upload-keystore.jks
  build-args: --release
  whatsnew-directory: store/whatsnew
```

### Artifacts

Cada release guarda el AAB y los símbolos de ofuscación como artifact durante 30 días. Los símbolos sirven para desofuscar trazas de crashes.
