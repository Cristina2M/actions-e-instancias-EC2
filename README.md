# actions-e-instancias-EC2

## 1. Creación y configuración de la instancia EC2

### 1.1 Lanzar la instancia
- En **AWS Academy Learner Lab**, abrí el **AWS Management Console**.
- Fui a **EC2 → Launch Instance**.
- Seleccioné:
  - **Amazon Linux 2 AMI**
  - **t2.micro**
  - **Par de claves**: descargué un archivo `labuser.pem` (clave privada SSH).
- En el campo **User Data**, añadí este script para instalar y configurar Nginx automáticamente:

```bash
#!/bin/bash
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
```

* En **Security Group**, habilité:

  * Puerto **22 (SSH)**
  * Puerto **80 (HTTP)**

---

## 2. Conexión a la instancia EC2

Para conectarme por SSH, ejecuté:

```bash
ssh -i "labuser.pem" ec2-user@<IP_PUBLICA>
```

🔸 Error inicial:

```
Warning: Identity file labuser (1).pem not accessible: No such file or directory.
```

➡️ **Solución:** Asegurarme de estar en el mismo directorio que el archivo `.pem` y corregir el nombre (quitando el espacio o paréntesis).

🔸 Luego tuve:

```
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

➡️ **Solución:** Verifiqué que el usuario fuera `ec2-user` y que el archivo `.pem` tuviera permisos correctos:

```bash
chmod 600 labuser.pem
```

Una vez dentro, comprobé que Nginx funcionaba:

```bash
sudo systemctl status nginx
```

---

## 3. Configuración del repositorio y estructura

* Reutilicé el mismo proyecto del **Ejercicio 1 (S3)**.
* Estructura base:

```
src/
├── index.html
├── style.css
├── app.js
test/
├── app.test.js
README.md
```

* En **GitHub**, configuré los siguientes **Secrets**:

| Nombre del Secret       | Descripción                                                             |
| ----------------------- | ----------------------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | ID de acceso desde el archivo de credenciales del Lab                   |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta                                                           |
| `AWS_SESSION_TOKEN`     | Token temporal                                                          |
| `AWS_REGION`            | Región (`us-east-1`)                                                |
| `EC2_HOST`              | IP pública de la instancia EC2 (3.84.125.23)                             |
| `EC2_USER`              | `ec2-user`                          |
| `EC2_KEY`               | Contenido completo del archivo `.pem`    |

---

## 4. Pruebas unitarias y documentación

### 4.1 Pruebas con Jest

Ejecuté las pruebas con:

```bash
npm test
```

🔹 Error inicial:

```
ReferenceError: document is not defined
```

➡️ **Solución:** Aislar la lógica del DOM y configurar Jest con `testEnvironment: "jsdom"` en `jest.config.js`.

---

### 4.2 Documentación con JSDoc

Ejecuté:

```bash
npm run docs
```

🔹 Error inicial:

```
FATAL: Unable to load template: Cannot find module 'default/publish'
```

➡️ **Solución:** Borrar el campo `"template": "default"` del archivo `jsdoc.json`.

---

## 5. Workflow de GitHub Actions

Archivo: `.github/workflows/deploy-ec2.yml`

```yaml
name: Deploy Web to EC2

on:
  push:
    branches:
      - dev
      - main
      - 'feature/**'

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build site
        run: npm run predeploy || npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-dist
          path: dist

  deploy-to-ec2:
    name: Deploy to EC2
    runs-on: ubuntu-latest
    needs: build-and-test

    steps:
      - uses: actions/checkout@v4

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-dist
          path: dist

      - name: Create PEM key
        run: |
          echo "${{ secrets.EC2_KEY }}" > private_key.pem
          chmod 600 private_key.pem

      - name: Copy files to EC2
        run: |
          scp -o StrictHostKeyChecking=no -i private_key.pem -r dist/* ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/home/ec2-user/app/

      - name: Move files to Nginx and restart service
        run: |
          ssh -o StrictHostKeyChecking=no -i private_key.pem ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} "sudo rm -rf /usr/share/nginx/html/* && sudo mv /home/ec2-user/app/* /usr/share/nginx/html/ && sudo systemctl restart nginx"
```

---

## 6. Errores encontrados y soluciones

| Error                                                              | Causa                                                  | Solución                                                                            |
| ------------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `scp: stat local "dist/*": No such file or directory`              | No se había generado el build                          | Ejecutar `npm run predeploy` antes del deploy                                       |
| `scp: dest open "/usr/share/nginx/html/app.js": Permission denied` | El usuario `ec2-user` no tenía permisos en esa carpeta | Solucionado moviendo los archivos primero a `/home/ec2-user/app` y luego con `sudo` |
| `Artifact not found for name: build-dist`                          | El job anterior no subía ningún artifact               | Añadido el paso `upload-artifact` en el workflow                                    |
| `403: Permission denied to github-actions[bot]`                    | GitHub Actions no tenía permiso para hacer push        | Añadido `permissions: write-all` o desactivado el push automático de docs           |

---

## 7. Comprobación del despliegue

1. Confirmé que el workflow en **GitHub Actions** terminó correctamente (todo en verde ✅).
2. Me conecté por SSH:

   ```bash
   ssh -i "labuser.pem" ec2-user@3.84.125.23
   ```
3. Revisé que los archivos estuvieran en `/usr/share/nginx/html/`:

   ```bash
   ls -l /usr/share/nginx/html
   ```
4. Verifiqué el servicio Nginx:

   ```bash
   sudo systemctl status nginx
   ```

   → Estado: **active (running)**
5. Accedí desde el navegador a:

   ```
   http://3.84.125.23
   ```

   y pude ver la aplicación web funcionando






