# Tutorial de ingreso y Uso del Cluster QB
## 1. Armado de Clave SSH

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

## 3. Configuración del Archivo `.ssh/config`

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

Reemplaza `minombre` con tu nombre de usuario en el cluster.

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

## 5. Tu Home en el Cluster

Tu directorio de inicio se encuentra en:

```bash
cd ~
pwd
```

Aquí puedes almacenar tus scripts, datos y resultados.

## 6. Instalación de software y paquetes

No hay permisos para usar `sudo` en el cluster, por lo que no podrás instalar paquetes de sistema.

Sugerencias:
- Verifica primero si la herramienta ya existe en los módulos del cluster o en carpetas compartidas (por ejemplo `/grupos/...`).
- Para software de usuario, instala todo con `conda`/`miniconda` dentro de tu home. 

**https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html**

Luego puedes complementar con `pip` para paquetes de Python que no estén en los canales de conda.
- Mantén ambientes separados para distintos proyectos (`conda create -n proyecto python=3.11`).
- Si una dependencia no está en conda, instala desde fuente bajo tu home y añade las rutas con variables de entorno (`PATH`, `LD_LIBRARY_PATH`, etc.).

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

Usá `Ctrl+A` seguido de `D` para desprenderte de la sesión de `screen` y `screen -r descarga` para retomarla.

## 8. Nodo cranex - NO USAR para Trabajar

⚠️ **IMPORTANTE**: El nodo `cranex` es el login node (cabecera del cluster).

**NO debes usar este nodo para ejecutar trabajos**, ya que consume recursos compartidos.

Siempre debes solicitar un nodo de cómputo con CPU disponible usando `srun` o `sbatch`.

## 9. Instalar y Configurar Screen

Para mantener sesiones persistentes, instala `screen` o usa directamente si ya está disponible:

```bash
screen --version
```

## 10. Crear Screen y Pedir un Nodo

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

## 11. Envío de Trabajos y Comandos Útiles

### Enviar un trabajo con `sbatch`

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
#SBATCH --nodelist=nodo2               # Nodo específico (opcional)
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

4. **Desprenderse de screen:**
   - Presiona `Ctrl+A` seguido de `D`

5. **Reconectar a screen:**
   ```bash
   screen -r trabajo
   ```



