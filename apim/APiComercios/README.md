**APi Comercios**

- **Descripción:** API proxy demostrativa para APIM que reenvía llamadas al backend de disponibilidad de farmacias (http://4.157.52.221/api).

Importar manualmente en APIM (Azure CLI):

```bash
# Variables
RG=my-resource-group
APIM_NAME=my-apim-service
# Importar OpenAPI y asignar path `api-comercios`
az apim api import --resource-group $RG --service-name $APIM_NAME --path api-comercios --api-id api-comercios --specification-format OpenApi --specification-path apim/APiComercios/openapi.yaml --service-url http://4.157.52.221/api
```

Agregar policy (ejemplo) desde archivo:

```bash
# Sube la policy XML definida en apim/APiComercios/policy.xml
# Usando REST (ejemplo):
SUBSCRIPTION_ID=<your-subscription>
API_ID=api-comercios
az rest --method PUT \
  --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.ApiManagement/service/$APIM_NAME/apis/$API_ID/policy?api-version=2021-08-01" \
  --body "{\"properties\":{\"value\":\"<policies>...your policy xml here...</policies>\",\"format\":\"rawxml\"}}"
```

GitHub Actions (simple import pipeline):

```yaml
name: Deploy API to APIM
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Azure CLI login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - name: Import API into APIM
        run: |
          az apim api import --resource-group ${{ env.AZ_RG }} --service-name ${{ env.APIM_NAME }} --path api-comercios --api-id api-comercios --specification-format OpenApi --specification-path apim/APiComercios/openapi.yaml --service-url http://4.157.52.221/api
        env:
          AZ_RG: ${{ secrets.AZURE_RESOURCE_GROUP }}
          APIM_NAME: ${{ secrets.AZURE_APIM_NAME }}
```

Notas:
- Reemplaza variables por tus valores.
- Para demostración de CI/CD, empuja `openapi.yaml` al repo; la pipeline importará/actualizará la API en APIM automáticamente.
- La asignación a un *product* (p. ej. `comercios`) no requiere cambios en el `openapi.yaml`. Los *products* y las políticas se administran por separado en APIM.

GitHub Actions / Secrets:
- Crea un *secret* llamado `AZURE_CREDENTIALS` con JSON de un service principal (usa `az ad sp create-for-rbac --name "github-actions-apim" --role "Contributor" --scopes /subscriptions/<sub_id>/resourceGroups/<rg> --sdk-auth` y copia el JSON de salida).
- Opcional: puedes añadir `AZURE_RESOURCE_GROUP`, `AZURE_APIM_NAME` y `AZURE_SUBSCRIPTION_ID` a Secrets o dejar las variables `env` en el workflow (el workflow ya incluye valores por defecto para tu APIM demo).

Push to GitHub (if not already pushed):

```bash
# From repository root
git init
git add .
git commit -m "Add APi Comercios OpenAPI, policy and CI workflow"
git remote add origin https://github.com/jimmypadillabit/ApiProxyAPIM.git
git branch -M main
git push -u origin main
```

Cómo funciona el workflow `.github/workflows/deploy-apim.yml`:
- Verifica si existe una API con `path` = `api-comercios` y aborta si hay colisión.
- Importa/actualiza la API con `az apim api import`.
- Crea el product `comercios` si no existe y asigna la API al product.
- Aplica la `policy.xml` (PUT REST a la API policy) y realiza una llamada rápida de comprobación.

Seguridad y consideraciones:
- El workflow usa `AZURE_CREDENTIALS` (service principal); no pongas credenciales con más permisos que los necesarios.
- Asegúrate de que `api-id` y `path` (`api-comercios`) no coincidan con la API ya desplegada que deseas conservar (para evitar sobrescrituras accidentales).

Puedo:
- Añadir un paso para validar la respuesta de la API y publicar el resultado como artefacto de la ejecución.
- Ayudarte a crear el `AZURE_CREDENTIALS` si quieres que lo haga desde tu CLI (requiere permiso para crear SP).
