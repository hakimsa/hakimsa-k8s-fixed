# 🔧 Proyecto Kubernetes ArgoCD + Image Updater - ARREGLADO

## ❌ Problema Original

El proyecto tenía el siguiente problema: **ambos entornos (dev y prod) se actualizaban con la misma imagen**.

### Causas identificadas:

1. **ImageUpdater CRD mal configurados**: Los archivos `flutter-web-image-updater-dev.yaml` y `flutter-web-image-updater-prod.yaml` usaban el mismo `name: hakimsamouh/flutter-web` en `manifestTargets`, lo que causaba que ambos escribieran en el mismo lugar.

2. **Mezcla de métodos**: Se usaban tanto ImageUpdater CRD como anotaciones en las Applications, causando conflictos.

3. **Falta de especificación de branch**: No se indicaba en qué branch de Git escribir los cambios.

## ✅ Solución Implementada

### Cambios realizados:

1. **Eliminados los ImageUpdater CRD**: Ya no se usan `flutter-web-image-updater-dev.yaml` ni `flutter-web-image-updater-prod.yaml`

2. **Configuración mediante anotaciones**: Todo se configura ahora en las Applications usando anotaciones de ArgoCD Image Updater

3. **Separación por branches**:
   - **DEV**: Actualiza en branch `develop` → path `deploy-front/dev`
   - **PROD**: Actualiza en branch `main` → path `deploy-front/prod`

### Estructura corregida:

```
k8s-fixed/
├── applications/
│   ├── flutter-web-dev.yaml   ← Configurado para branch develop
│   └── flutter-web-prod.yaml  ← Configurado para branch main
└── deploy-front/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    ├── dev/
    │   └── kustomization.yaml  ← Se actualizará solo con tags dev-*
    └── prod/
        └── kustomization.yaml  ← Se actualizará solo con tags prod-*
```

## 🎯 Cómo funciona ahora

### Para DEV:
1. Push a `develop` → CI/CD construye imagen `hakimsamouh/flutter-web:dev-X.X.X-HASH`
2. ArgoCD Image Updater detecta nueva imagen con tag `dev-*`
3. Actualiza `deploy-front/dev/kustomization.yaml` en branch `develop`
4. Commit automático a `develop`
5. ArgoCD sincroniza cambios al namespace `dev`

### Para PROD:
1. Merge a `main` → CI/CD construye imagen `hakimsamouh/flutter-web:prod-X.X.X-HASH`
2. ArgoCD Image Updater detecta nueva imagen con tag `prod-*`
3. Actualiza `deploy-front/prod/kustomization.yaml` en branch `main`
4. Commit automático a `main`
5. ArgoCD sincroniza cambios al namespace `prod`

## 📋 Anotaciones clave en Applications

```yaml
annotations:
  # Lista de imágenes a monitorear
  argocd-image-updater.argoproj.io/image-list: flutter-web=hakimsamouh/flutter-web
  
  # Filtro de tags (dev-* o prod-*)
  argocd-image-updater.argoproj.io/flutter-web.allow-tags: regexp:^dev-.*
  
  # Estrategia: usar la más reciente
  argocd-image-updater.argoproj.io/flutter-web.update-strategy: newest-build
  
  # Nombre de la imagen en kustomization
  argocd-image-updater.argoproj.io/flutter-web.kustomize.image-name: hakimsamouh/flutter-web
  
  # Método de escritura: Git con credenciales
  argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
  
  # Branch donde escribir (develop para dev, main para prod)
  argocd-image-updater.argoproj.io/git-branch: develop
```

## 🚀 Despliegue

### 1. Crear secret con credenciales de Git (si no existe):

```bash
kubectl create secret generic git-creds \
  --from-literal=username=<TU_USUARIO_GITHUB> \
  --from-literal=password=<TU_TOKEN_GITHUB> \
  -n argocd
```

### 2. Aplicar las Applications:

```bash
kubectl apply -f k8s-fixed/applications/flutter-web-dev.yaml
kubectl apply -f k8s-fixed/applications/flutter-web-prod.yaml
```

### 3. Verificar que funciona:

```bash
# Ver logs del image updater
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# Ver estado de las apps
kubectl get applications -n argocd

# Ver si se actualizan los tags
kubectl get application flutter-web-dev -n argocd -o jsonpath='{.status.summary.images}'
kubectl get application flutter-web-prod -n argocd -o jsonpath='{.status.summary.images}'
```

## 🔍 Troubleshooting

### Si no se actualizan las imágenes:

1. **Verificar que el image updater está corriendo**:
   ```bash
   kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater
   ```

2. **Ver logs del image updater**:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=100
   ```

3. **Verificar que las credenciales de Git son correctas**:
   ```bash
   kubectl get secret git-creds -n argocd -o yaml
   ```

4. **Forzar actualización manual**:
   ```bash
   # Añadir anotación para forzar actualización
   kubectl annotate application flutter-web-dev -n argocd \
     argocd-image-updater.argoproj.io/image-list=flutter-web=hakimsamouh/flutter-web --overwrite
   ```

### Si dev y prod siguen usando la misma imagen:

- Verifica que cada entorno está apuntando al branch correcto en `source.targetRevision`
- Confirma que los tags de las imágenes siguen el patrón `dev-*` y `prod-*`
- Revisa que los kustomization.yaml en cada branch (develop/main) tienen tags diferentes

## 📝 Notas importantes

- **Branches separados**: Es crucial que `develop` y `main` tengan archivos `kustomization.yaml` independientes
- **Tags con prefijo**: Las imágenes deben tener prefijo `dev-` o `prod-` para que el filtro funcione
- **Write-back method**: Se usa `git:secret` para que el image updater pueda hacer commits automáticos
- **No más ImageUpdater CRD**: Se eliminaron porque causaban conflictos con las anotaciones

## 🎉 Resultado

Ahora cada entorno (dev y prod) se actualiza de forma independiente:
- DEV solo con imágenes `dev-*` en branch `develop`
- PROD solo con imágenes `prod-*` en branch `main`

**No habrá más sobrescritura entre entornos.**
