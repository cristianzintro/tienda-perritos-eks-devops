Tienda Perritos EKS

Aplicación de tienda de alimentos para perritos: frontend, backend
(Node.js + Express) y base de datos (MySQL), desplegada en AWS.

Integrantes

- Cristian Alvarez B
- Cristian Caceres G

Arquitectura

| COMPONENTE        | TECNOLOGIA        | PUERTO 
| Frontend          |  Nginx + HTML/JS  | 80 
| Backend           | Node.js + Express | 3001 
| Base de Datos     | MySQL 8           | 3306 

Estrategia: GitFlow

Utilizamos GitFlow para organizar el trabajo en equipo de forma ordenada:

- main: Rama principal que contiene la versión estable lista para producción.
- develop: Rama de integración donde se unen todas las nuevas características antes de pasar a producción.
- feature/nombre: Ramas temporales creadas desde "develop" para desarrollar nuevas funciones.
- hotfix/nombre: Ramas temporales creadas desde "main" para solucionar errores urgentes en producción.

Convenciones de commits

Seguimos un formato estándar y claro para los mensajes en los commits creados:

Estructura:

| Prefijo      | Cuándo usarlo                           | Ejemplo 
| feat         | Nueva funcionalidad                     | feat: agregar pasarela de pagos 
| fix          | Corrección de errores (bugs)            | fix: corregir cálculo de stock` 
| docs         | Cambios en la documentación             | docs: actualizar el readme
| ci           | Cambios en el pipeline o automatización | ci: configurar github actions
| refactor`    | Mejora de código sin cambiar su función | refactor: optimizar conexion a base de datos 

Flujo de merge (pull requests)

1. Desde develop se crea la rama feature/nombre de integrante.
2. Se desarrolla y se commitea con la convención.
3. Se abre un PR hacia develop describiendo el cambio.
4. El CI debe quedar en verde antes de fusionar.
5. La pareja revisa, comenta si hace falta y aprueba.
6. Se fusiona con merge commit y se elimina la rama.
7. Releases: PR de develop hacia main. Urgencias: hotfix desde main hacia main y develop.

Estrategia de revisión

- Todo cambio entra por pull request; nunca push directo a main ni develop.
- Mínimo un revisor (la pareja).
- Lista de chequeo: probado, no rompe lo existente, commit correcto, sin secretos.

Pipeline CI/CD (GitHub Actions)

- .github/workflows/ci.yml → corre en cada push a develop y en cada pull request hacia main.
  Valida sintaxis del backend (npm test), del frontend (node --check) y estructura de carpetas.

- .github/workflows/deploy-ec2.yml → corre en push a main y manual: prueba backend y despliega
  a EC2 vía SSM. Requiere secrets AWS (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN).

- deploy-eks.yml.disabled → despliegue a EKS, deshabilitado.

Cómo correr el proyecto

    docker compose up -d --build
    backend: cd backend && npm install && npm test

Problemas encontrados y soluciones
1. Nuestro pipeline falló por falta de AWS Secrets ya que se uso un repo con esas ruta de aws, pero para esta evaluacion no fue necesario usarlo.
2. Pod no tomaba la imagen - Solucion :latest : kubectl rollout restart.
3. Historial mostraba commits del repo original - Solucion: rama orphan con historial limpio.

Aplicación de IA

IA:  Gemini
     Chat GPT

Reflexiones personales

Cristian Caceres G:
Como equipo esta evaluacion fue una gran desafio, ya que con mi compañero alfin pudimos trabajar desde diferentes ramas integrando estrategicamente cada cambio con su respectivo mensaje, para que no haya ningun cambio sin documentar, como tambien los errores sean vistos. 

Cristian Alvarez B:
Gracias a la ayuda de la IA las cuales nos sirven de soporte pudimos llevar una estructura sencilla, y sin caer en errores de sintaxis a la hora de ejecutar los comandos "git". Tambien nos ayuda a tener un flujo constante dentro de nuestro proyecto sin caer en el plagio.

