# Tutorial de ingreso y uso del Cluster QB

Esta guía resume el flujo básico para empezar a usar el cluster QB sin romper nada importante en el primer día.

Documentación oficial del cluster:
- [Página principal del Cluster QB](https://docs.cluster.qb.fcen.uba.ar/index.php/P%C3%A1gina_principal)

## Antes de empezar

Necesitas:
- Tener un usuario asignado en el cluster.
- Haber enviado tu clave pública a los organizadores.
- Conectarte desde tu máquina local al cluster por `ssh`.

Recuerda:
- El nodo de login (`cranex`) es solo para conectarte, mover archivos y preparar trabajos.
- Los cálculos y programas pesados deben correr en nodos de cómputo pedidos con `srun` o `sbatch`.
- En este cluster no hay `sudo` ni sistema de `modules`.

## 1. Armado de clave SSH

Genera un par de claves SSH en tu máquina local:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

Este comando crea:
- `~/.ssh/id_rsa` - Clave privada (NUNCA compartir)
- `~/.ssh/id_rsa.pub` - Clave pública (a compartir)

Puedes verificar las claves generadas:

```bash
ls -la ~/.ssh/
cat ~/.ssh/id_rsa.pub
```

## 2. Envío de Clave Pública a Organizadores

Envía el contenido de tu **clave pública** a los organizadores del cluster QB:

```bash
cat ~/.ssh/id_rsa.pub
```

Copia el contenido y envíalo por email/whatsapp. Los organizadores la añadirán a tu usuario en el cluster.

⚠️ **IMPORTANTE**: Nunca compartas tu clave privada (`id_rsa`)

Antes de seguir, espera la confirmación de que tu clave fue cargada en el cluster. Si intentas entrar antes de tiempo, es normal recibir un error como:

```bash
Permission denied (publickey)
```

## 3. Configuración del archivo `.ssh/config`

Crea o edita el archivo `~/.ssh/config` en tu máquina local para simplificar el acceso:

```bash
nano ~/.ssh/config
```

Añade la siguiente configuración:

```
Host qb
    HostName cluster.qb.fcen.uba.ar
    User minombre
```

Reemplaza `minombre` con tu nombre de usuario real en el cluster.

Ajusta permisos del archivo:

```bash
chmod 600 ~/.ssh/config
```

## 4. Acceso al Cluster

Una vez configurado, accede al cluster con:

```bash
ssh qb
```

O directamente:

```bash
ssh minombre@cluster.qb.fcen.uba.ar
```

En este ejemplo, reemplaza `minombre` por tu usuario real.

## 5. Tu Home en el Cluster

Tu directorio de inicio se encuentra en:

```bash
cd ~
pwd
```

Aquí puedes almacenar tus scripts, datos y resultados.

Para confirmar dónde estás parado al entrar:

```bash
whoami
hostname
pwd
```

Si `hostname` muestra `cranex`, estás en el nodo de login. Aún no estás en un nodo de cómputo.

## 6. Instalación de software y paquetes

No hay permisos para usar `sudo` en el cluster, por lo que no podrás instalar paquetes de sistema.

Sugerencias:
- Verifica primero si la herramienta o los datos ya existen en carpetas compartidas (por ejemplo `/grupos/...`).
- Este cluster no usa un sistema de `modules`. Si necesitas software adicional, instálalo en tu home.
- Para software de usuario, instala todo con `conda`/`miniconda` dentro de tu home.

[Guía de instalación de Miniconda/Conda para Linux](https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html)

Luego puedes complementar con `pip` para paquetes de Python que no estén en los canales de conda.

- Mantén ambientes separados para distintos proyectos (`conda create -n proyecto python=3.11`).
- Si una dependencia no está en conda, instálala manualmente desde fuente bajo tu home y añade las rutas con variables de entorno (`PATH`, `LD_LIBRARY_PATH`, etc.).
- Si instalas Miniconda en tu home, recuerda inicializarlo antes de usarlo:

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate mi_entorno
```

Una práctica razonable es organizar tu home así:

```bash
~/miniconda3
~/proyectos
~/software
~/datos
```

## 7. ¿Qué hago si debo descargar un programa o datos?

Para descargas largas o que no pueden interrumpirse, iniciá una sesión de `screen` y realizá la transferencia dentro de ella:

```bash
ssh qb
screen -S descarga
wget -c https://servidor/archivo.tar.gz
```

Recordá que en la red de la FCEN solo están habilitados los puertos estándar: HTTP 80, HTTPS 443, FTP 21 y SSH 22. Si el origen usa otros puertos deberás descargar desde otro lugar (tu máquina o una VM externa) y luego copiar los datos al cluster.

Antes de bajar grandes bases de datos, revisá `/home/grupos`, donde ya hay muchos datasets disponibles para reutilizar.

Comandos útiles:

Descargar archivos grandes con reintentos y reanudación:

```bash
wget -c https://ejemplo.com/datos.tar.gz          # -c reanuda si se corta
wget --mirror --continue --no-parent URL_BASE     # Mirroring limitado
```

Sincronizar carpetas entre tu PC y el cluster con `rsync`:

```bash
# De tu PC al cluster
rsync -avhP ./proyecto/ usuario@cluster.qb.fcen.uba.ar:~/proyecto/

# Del cluster a tu PC
rsync -avhP usuario@cluster.qb.fcen.uba.ar:~/resultados/ ./resultados_local/
```

Reemplaza `usuario` por tu usuario real en el cluster.

Usá `Ctrl+A` seguido de `D` para desprenderte de la sesión de `screen` y `screen -r descarga` para retomarla.

## 8. Nodo cranex - NO USAR para Trabajar

⚠️ **IMPORTANTE**: El nodo `cranex` es el login node (cabecera del cluster).

**NO debes usar este nodo para ejecutar trabajos**, ya que consume recursos compartidos.

Siempre debes solicitar un nodo de cómputo con CPU disponible usando `srun` o `sbatch`.

## 9. Usar `screen` para sesiones persistentes

Para mantener sesiones persistentes, verifica primero si `screen` ya está disponible:

```bash
screen --version
```

Comandos útiles:

```bash
screen -S mi_sesion    # Crear una sesión nueva
screen -ls             # Listar sesiones existentes
screen -r mi_sesion    # Reconectarse a una sesión
exit                   # Cerrar la shell actual
```

## 10. Crear una sesión de `screen` y pedir un nodo

Conecta al cluster y crea una nueva sesión de `screen`:

```bash
ssh qb
screen -S mi_sesion
```

Dentro de la sesión de screen, solicita un nodo de cómputo:

```bash
srun --nodes=1 --ntasks-per-node=1 --cpus-per-task=1 -p cpu --pty bash -i
```

Desglose del comando:
- `--nodes=1` - Usa 1 nodo
- `--ntasks-per-node=1` - 1 tarea por nodo
- `--cpus-per-task=1` - 1 CPU por tarea
- `-p cpu` - Partición CPU (puedes usar otras como `gpu`, etc.)
- `--pty bash -i` - Abre una shell interactiva

Una vez ejecutado, tendrás acceso a un nodo de cómputo. Puedes desprenderte de la sesión con `Ctrl+A+D`.

Para verificar que ya no estás en el login node:

```bash
hostname
```

El nombre del host debería ser distinto de `cranex`.

## 11. Envío de Trabajos y Comandos Útiles

### Enviar un trabajo con `sbatch`

Los ejemplos de esta sección usan nombres de usuario, rutas y ambientes ficticios. Antes de ejecutar, reemplaza:
- `usuario` por tu usuario real.
- `/home/usuario/...` por tus rutas reales.
- `mi_env_cpu` y `mi_env_gpu` por los nombres reales de tus ambientes.
- Scripts, datasets y parámetros por los de tu proyecto.

Crea un script `mi_trabajo_cpu.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=analisis_cpu        # Nombre del trabajo
#SBATCH --nodes=1                      # Todos los procesos en un nodo
#SBATCH --ntasks=1                     # Una sola tarea
#SBATCH --cpus-per-task=4              # Núcleos por tarea
#SBATCH --partition=cpu                # Partición CPU
#SBATCH --time=02:00:00                # Tiempo máximo
#SBATCH --chdir=/home/usuario/proyecto # Directorio donde se ejecuta
#SBATCH --output=cpu_%j.out            # Log de salida
#SBATCH --error=cpu_%j.err             # Log de error

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    partición: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
date +"inicio %F - %T"

echo "\n----------------------------------------------\n"

# Inicializa y activa el ambiente de conda
eval "$(/home/usuario/miniconda3/bin/conda shell.bash hook)"
conda activate mi_env_cpu

# Variables de entorno útiles para el proyecto
export PROJECT_HOME="/home/usuario/proyecto"
export PYTHONPATH="$PROJECT_HOME/src:$PYTHONPATH"

# SLURM_CPUS_PER_TASK lo setea el scheduler según el valor pedido en --cpus-per-task
python "$PROJECT_HOME/scripts/procesar.py" --input datos/input.csv --threads ${SLURM_CPUS_PER_TASK}

echo "\n----------------------------------------------\n"
sacct --format=JobID,Submit,Start,End,State,Partition,ReqTRES%30,CPUTime,MaxRSS,NodeList --units=M -j $SLURM_JOB_ID
```

Para trabajos que requieren GPU crea `mi_trabajo_gpu.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=modelo_gpu          # Nombre del trabajo
#SBATCH --nodes=1                      # Un nodo
#SBATCH --ntasks=1                     # Una tarea
#SBATCH --cpus-per-task=8              # CPUs para preparar datos
#SBATCH --partition=gpu                # Partición GPU
#SBATCH --gres=gpu:1                   # Cantidad de GPUs
#SBATCH --time=04:00:00                # Tiempo máximo
#SBATCH --chdir=/home/usuario/modelos  # Directorio de trabajo
#SBATCH --output=gpu_%j.out            # Log de salida
#SBATCH --error=gpu_%j.err             # Log de error

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    partición: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
date +"inicio %F - %T"

echo "\n----------------------------------------------\n"

eval "$(/home/usuario/miniconda3/bin/conda shell.bash hook)"
conda activate mi_env_gpu

export DATASETS="/grupos/proyecto/datasets"
export CHECKPOINT_DIR="$PWD/modelos"
mkdir -p "$CHECKPOINT_DIR"

python entrenar.py \
   --data "$DATASETS/corpus" \
   --output "$CHECKPOINT_DIR/exp_${SLURM_JOB_ID}" \
   --batch 32 --epochs 50 --use-gpu

echo "\n----------------------------------------------\n"
sacct --format=JobID,Submit,Start,End,State,Partition,ReqTRES%30,CPUTime,MaxRSS,NodeList,MaxVMSize --units=M -j $SLURM_JOB_ID
```

Envía cualquiera de los trabajos:

```bash
sbatch mi_trabajo_cpu.sh
sbatch mi_trabajo_gpu.sh
```

### Comandos Útiles

**Editar archivos de texto:**

```bash
nano archivo.txt                # Editar un archivo en terminal
nano script.py                  # Abrir o crear un script
```

Atajos útiles en `nano`:
- `Ctrl+O` guarda los cambios.
- `Ctrl+X` sale del editor.
- `Ctrl+K` corta una línea.
- `Ctrl+U` pega la última línea cortada.

**Ver archivos y buscar contenido:**

```bash
cat archivo.txt                 # Mostrar un archivo completo
less archivo_grande.txt         # Ver un archivo largo por páginas
head archivo.txt                # Primeras líneas
tail archivo.txt                # Últimas líneas
tail -f logfile.out             # Seguir un log en tiempo real
grep "patron" archivo.txt       # Buscar texto dentro de un archivo
grep -R "patron" carpeta/       # Buscar texto dentro de una carpeta
```

**Inspeccionar tablas y datos tabulares:**

```bash
column -t -s, archivo.csv       # Alinear columnas simples separadas por comas
```

Si `visidata` está instalado en tu entorno o en tu ambiente de `conda`, puede ser muy útil para explorar tablas grandes en terminal:

```bash
vd archivo.csv
vd archivo.tsv
vd resultados.xlsx
```

Si todavía no lo tienes, puedes instalarlo en tu ambiente con:

```bash
conda install -c conda-forge visidata
```

**Mover, copiar y organizar archivos:**

```bash
ls                              # Listar archivos
ls -lh                          # Listar con tamaños legibles
pwd                             # Mostrar directorio actual
mkdir mi_carpeta                # Crear una carpeta
mkdir -p resultados/figuras     # Crear carpetas anidadas
cp archivo.txt copia.txt        # Copiar un archivo
cp -r carpeta/ carpeta_backup/  # Copiar una carpeta completa
mv viejo.txt nuevo.txt          # Renombrar o mover archivo
mv archivo.txt carpeta/         # Mover un archivo a otra carpeta
rm archivo.txt                  # Borrar un archivo
rm -r carpeta/                  # Borrar una carpeta y su contenido
```

⚠️ Usa `rm` con cuidado: en terminal no hay papelera.

**Revisar espacio en disco:**

```bash
du -sh .                        # Tamaño total de la carpeta actual
du -sh *                        # Tamaño de cada archivo/carpeta
df -h                           # Espacio disponible en los discos montados
quota -s                        # Cuota personal, si está configurada
```

**Ver colas y trabajos:**

```bash
squeue                          # Ver todos los trabajos
squeue -u $USER                 # Ver solo mis trabajos
squeue -j JOBID                 # Ver detalles de un trabajo específico
```

**Ver información del cluster:**

```bash
sinfo                           # Ver nodos y particiones disponibles
sinfo -p cpu                    # Ver estado de partición específica
sinfo -N                        # Ver nodos con más detalles
```

**Gestionar trabajos:**

```bash
scancel JOBID                   # Cancelar un trabajo
scancel -u $USER                # Cancelar todos mis trabajos
sstat -j JOBID                  # Ver estadísticas de un trabajo en ejecución
sacct -j JOBID                  # Ver información histórica de un trabajo
```

**Información de particiones:**

```bash
sinfo --format="%20N %10c %10m %20A"
```

## 12. Recursos

Resumen de nodos y hardware reportado del cluster:

| Nombre | Procesador | CPUs | Memoria | GPU(s) | CUDA | Driver |
| --- | --- | ---: | --- | --- | --- | --- |
| nodo1 | RYZEN 9 5900X | 24 | 62 Gb | 1 x geforce_gtx_1080ti | 12.4 | 570.133.20 |
| nodo2 | I7-8700 | 12 | 62 Gb | 2 x geforce_gtx_1080ti | 12.4 | 570.144 |
| nodo3 | I7-8700 | 12 | 62 Gb | 2 x geforce_rtx_2080 | 12.4 | |
| nodo4 | I7-8700 | 12 | 62 Gb | 2 x geforce_rtx_2080 | 10.1 | |
| nodo5 | I7-8700 | 12 | 46 Gb | 2 x geforce_rtx_2080 | 12.4 | 570.133.07 |
| nodo6 | I7-8700 | 12 | 62 Gb | 2 x geforce_rtx_2080 | 12.4 | 570.124.04 |
| nodo7 | I7-8700 | 12 | 62 Gb | 1 x geforce_rtx_2080 | 12.4 | 570.133.20 |
| nodo8 | I7-8700 | 12 | 62 Gb | 2 x geforce_rtx_2080 | 12.4 | 560.35.03 |
| nodo9 | I7-8700 | 12 | 62 Gb | 1 x geforce_rtx_2080 | 12.4 | 570.133.20 |
| nodo10 | RYZEN 9 5900X | 24 | 62 Gb | 1 x geforce_gtx_1080ti | | 570.133.20 |
| nodo11 | RYZEN 9 5900X | 24 | 62 Gb | 1 x NVIDIA GeForce RTX 3050 | 12.8 | |
| nodo12 | RYZEN 9 5900X | 24 | 46 Gb | 1 x Geforce GTX 1050 | 12.6 | |
| nodo13 | RYZEN 9 5900X | 24 | 62 Gb | 1 x NVIDIA GeForce RTX 3050 | 10.1 | |
| nodo14 | RYZEN 9 5950X | 32 | 62 Gb | 1 x NVIDIA GeForce GT 630 | | |
| nodo15 | RYZEN 7 5700G | 16 | 62 Gb | 0 | | |
| sauron | RYZEN 7 3800X | 16 | 31 Gb | 1 x NVIDIA GeForce GTX 770 | 10.1 | |

Ten en cuenta:
- La disponibilidad real puede cambiar según mantenimiento, uso y configuración del scheduler.
- Algunas versiones de CUDA o drivers no figuran en todos los nodos; conviene verificarlo al iniciar un trabajo sensible a versiones.
- Si necesitas confirmar el estado actual de particiones y nodos, usa `sinfo` y `squeue`.

## Resumen de Flujo Típico

1. **Primer acceso:**
   ```bash
   ssh qb
   ```

2. **En una sesión persistente:**
   ```bash
   screen -S trabajo
   srun --nodes=1 --ntasks-per-node=1 --cpus-per-task=4 -p cpu --pty bash -i
   ```

3. **Ejecutar código:**
   ```bash
   cd ~/mi_proyecto
   python script.py
   ```

4. **Salir del nodo de cómputo cuando terminas:**
   ```bash
   exit
   ```

5. **Desprenderse de `screen`:**
   - Presiona `Ctrl+A` seguido de `D`

6. **Reconectar a `screen`:**
   ```bash
   screen -r trabajo
   ```

## Problemas frecuentes

**`Permission denied (publickey)`**

Tu clave pública todavía no fue cargada, estás usando otro usuario o tu `~/.ssh/config` quedó mal configurado.

**`screen: command not found`**

`screen` no está disponible en tu entorno actual. Consulta si está instalado en el cluster o usa una alternativa acordada por los organizadores.

**`srun` queda esperando recursos**

Es normal si no hay nodos libres. Puedes revisar el estado con:

```bash
squeue -u $USER
sinfo
```

**`conda: command not found`**

Miniconda no está inicializado en la shell actual. Ejecuta:

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate mi_entorno
```
