# Uso de FastTarget en el Cluster QB

Esta guía explica cómo ejecutar `FastTarget` en el cluster QB usando la instalación compartida del grupo y un ambiente `conda` propio de cada usuario.

## Ubicación del repositorio compartido

El código y las bases de datos están instalados en:

```bash
/grupos/Dario/fasttarget
```

En esta guía se usará esa ruta como `FASTTARGET_HOME`.

## Sincronizar el repositorio

Si el repositorio compartido ha sido actualizado, necesitas traer los cambios a tu copia local. Conectate al cluster y ejecuta:

```bash
ssh qb
cd /grupos/Dario/fasttarget
git pull
```

Es una buena práctica hacer esto regularmente para asegurarte de tener la versión más reciente del código y las bases de datos. Si hay cambios importantes, es posible que debas:
- Recrear tu ambiente `conda` o reinstalar algunos paquetes
- Actualizar las bases de datos si han sido modificadas

Para más información sobre actualizaciones y cambios en el repositorio, consulta:
- [Repositorio de FastTarget en GitHub](https://github.com/mcpalumbo/fasttarget)

## Antes de empezar

Necesitas:
- Tener acceso al cluster QB.
- Tener `miniconda` o `conda` instalado en tu `home`.
- Crear un directorio propio en tu `home` para guardar resultados y logs.

Por ejemplo:

```bash
mkdir -p ~/fasttarget_runs
mkdir -p ~/fasttarget_runs/logs
```

## 1. Crear tu ambiente `conda`

Cada usuario debe crear su propio ambiente `conda`. No uses el ambiente de otra persona.

Primero activa `conda` en tu shell:

```bash
source ~/miniconda3/etc/profile.d/conda.sh
```

Dirigete a la carpeta de Fasttarget para crear el ambiente:

```bash
cd /grupos/Dario/fasttarget
conda env create -f requirements.yml
conda activate fasttarget
```

## 2. Instalar Java y Nextflow en tu ambiente

Si vas a usar la versión con `nextflow`, instala también Java 17 y Nextflow dentro de tu entorno.

### Java 17

```bash
conda activate fasttarget
conda install -c conda-forge openjdk=17 -y
java -version
```

### Nextflow

Puedes seguir la documentación oficial:
- [Instalación de Nextflow](https://www.nextflow.io/docs/latest/install.html)

Una opción práctica es instalarlo dentro de tu `home` y dejarlo en tu `PATH`.

Por ejemplo:

```bash
mkdir -p ~/software/bin
cd ~/software/bin
curl -s https://get.nextflow.io | bash
chmod +x nextflow
```

Si `~/software/bin` no está en tu `PATH`, agrégalo temporalmente:

```bash
export PATH="$HOME/software/bin:$PATH"
```

O de forma persistente en `~/.bashrc`:

```bash
export PATH="$HOME/software/bin:$PATH"
```

Verifica que funcione:

```bash
nextflow -version
```

## 3. Variables útiles

Estas variables simplifican los comandos:

```bash
export FASTTARGET_HOME="/grupos/Dario/fasttarget"
export PATH="$FASTTARGET_HOME/ftscripts:$PATH"
```

Si usas `nextflow`, además conviene definir:

```bash
export JAVA_HOME="$HOME/miniconda3/envs/fasttarget"
export NXF_JAVA_HOME="$JAVA_HOME"
```

Puedes exportarlas en la sesión actual o dejarlas en `~/.bashrc` si vas a usar `FastTarget` con frecuencia.

## 4. Opción A: correr FastTarget sin `nextflow`

Esta opción ejecuta `fasttarget.py` directamente.

### Script de ejemplo para `sbatch`

Guarda este script como `fasttarget_directo.sh` en tu directorio de trabajo:

```bash
#!/bin/bash
#SBATCH --job-name=fasttarget
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=24
#SBATCH --partition=cpu
#SBATCH --chdir=/home/USUARIO/fasttarget_runs
#SBATCH --output=/home/USUARIO/fasttarget_runs/logs/fasttarget_%j.out
#SBATCH --error=/home/USUARIO/fasttarget_runs/logs/fasttarget_%j.err

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    particion: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
date +"inicio %F - %T"

echo "
--------------------------------------------------------------------------------
"

eval "$($HOME/miniconda3/bin/conda shell.bash hook)"
conda activate fasttarget

export FASTTARGET_HOME="/grupos/Dario/fasttarget"
export PATH="$FASTTARGET_HOME/ftscripts:$PATH"

OUTPUT_DIR="$HOME/fasttarget_runs/TEST_out_${SLURM_JOB_ID}"
mkdir -p "$OUTPUT_DIR"

python "$FASTTARGET_HOME/fasttarget.py" \
  --databases_path "$FASTTARGET_HOME/databases" \
  --output_path "$OUTPUT_DIR" \
  --config_file "$FASTTARGET_HOME/config_test.yml"


echo "
---------------------------------------------------------
"

echo
date +"fin %F - %T"
sacct --format=JobID,Submit,Start,End,State,Partition,ReqTRES%30,CPUTime,MaxRSS,NodeList --units=M -j ${SLURM_JOB_ID}
```

Antes de usarlo, reemplaza:
- `USUARIO` por tu nombre de usuario real.
- El archivo `config_test.yml` por el de tu organismo o corrida, si corresponde.

Luego envíalo con:

```bash
sbatch fasttarget_directo.sh
```

Los archivos definidos con:

```bash
#SBATCH --output=...
#SBATCH --error=...
```

guardan los logs del trabajo:
- El archivo `.out` contiene la salida estándar del job, es decir, mensajes informativos, impresión de progreso y cualquier salida normal del programa. Es lo que verías en la terminal.
- El archivo `.err` contiene la salida de error, incluyendo mensajes de error, warnings y trazas si algo falla.

Cuando una corrida no funciona como esperas, conviene revisar ambos archivos.

## 5. Opción B: correr FastTarget con `nextflow`

Esta opción usa el pipeline ubicado en:

```bash
/grupos/Dario/fasttarget/nextflow
```

### Script de ejemplo para `sbatch`

Guarda este script como `fasttarget_nextflow.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=NFfast
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --partition=cpu
#SBATCH --chdir=/grupos/Dario/fasttarget/nextflow
#SBATCH --output=/home/USUARIO/fasttarget_runs/logs/NFfasttarget_%j.out
#SBATCH --error=/home/USUARIO/fasttarget_runs/logs/NFfasttarget_%j.err

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    particion: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
date +"inicio %F - %T"

eval "$($HOME/miniconda3/bin/conda shell.bash hook)"
conda activate fasttarget

export FASTTARGET_HOME="/grupos/Dario/fasttarget"
export PATH="$FASTTARGET_HOME/ftscripts:$PATH"

export JAVA_HOME="$HOME/miniconda3/envs/fasttarget"
export PATH="$JAVA_HOME/bin:$PATH"
export NXF_JAVA_HOME="$JAVA_HOME"

export OUTPUT_PATH="$HOME/fasttarget_runs"
export NXF_TEMP="$OUTPUT_PATH/.nxf_tmp"
export TMPDIR="$NXF_TEMP"

mkdir -p "$NXF_TEMP"
mkdir -p "$OUTPUT_PATH/nextflow_run_${SLURM_JOB_ID}"

nextflow run main.nf \
  --databases_path "$FASTTARGET_HOME/databases" \
  --output_path "$OUTPUT_PATH/nextflow_run_${SLURM_JOB_ID}" \
  --config_file "config_nextflow_test.yml" \
  -profile slurm,singularity

echo
date +"fin %F - %T"
sacct --format=JobID,Submit,Start,End,State,Partition,ReqTRES%30,CPUTime,MaxRSS,NodeList --units=M -j ${SLURM_JOB_ID}
```

En este ejemplo se redirige el directorio temporal a:

```bash
$HOME/fasttarget_runs/.nxf_tmp
```

Esto se hace para que los archivos temporales de `nextflow` y de la corrida no intenten escribirse en ubicaciones con menos espacio o menos control por parte del usuario. Una vez que la corrida terminó y verificaste que los resultados finales están bien guardados, ese directorio temporal se puede borrar:

```bash
rm -r ~/fasttarget_runs/.nxf_tmp
```

Además, después de correr con `nextflow` suelen quedar dos carpetas dentro de:

```bash
/grupos/Dario/fasttarget/nextflow
```

Esas carpetas son:

```bash
.nextflow
work
```

- `.nextflow` guarda cache y metadatos de ejecución.
- `work` guarda archivos intermedios y logs de procesos.

Si vas a lanzar una corrida nueva desde cero, suele convenir borrarlas antes para evitar errores por estados previos o reutilización accidental de cache. La excepción es cuando quieres revisar logs, inspeccionar errores o entender qué pasó en la corrida anterior.

Por ejemplo:

```bash
cd /grupos/Dario/fasttarget/nextflow
rm -r .nextflow work
```

Antes de usarlo, reemplaza:
- `USUARIO` por tu usuario real.
- `config_nextflow_test.yml` por el archivo de configuración que quieras usar.

Luego envíalo con:

```bash
sbatch fasttarget_nextflow.sh
```

## 6. Ejecución interactiva

También puedes ejecutar `FastTarget` de forma interactiva para pruebas cortas:

```bash
ssh qb
screen -S fasttarget
srun --nodes=1 --ntasks=1 --cpus-per-task=4 -p cpu --pty bash -i
```

Dentro del nodo:

```bash
eval "$($HOME/miniconda3/bin/conda shell.bash hook)"
conda activate fasttarget

export FASTTARGET_HOME="/grupos/Dario/fasttarget"
export PATH="$FASTTARGET_HOME/ftscripts:$PATH"

mkdir -p ~/fasttarget_runs/test_interactivo

python "$FASTTARGET_HOME/fasttarget.py" \
  --databases_path "$FASTTARGET_HOME/databases" \
  --output_path "$HOME/fasttarget_runs/test_interactivo" \
  --config_file "$FASTTARGET_HOME/config_test.yml"
```

Para salir del nodo:

```bash
exit
```

Para desprenderte de `screen`:
- `Ctrl+A` seguido de `D`

## 7. Dónde quedan los resultados

Los resultados y logs deberían guardarse en un directorio propio en tu `home`, por ejemplo:

```bash
~/fasttarget_runs
~/fasttarget_runs/logs
```

Eso evita mezclar salidas entre usuarios y mantiene separado:
- el código compartido en `/grupos/Dario/fasttarget`
- tus salidas y logs en tu `home`

Si quieres compartirlos puedes mover la carpeta de resultados a `/grupos/Dario/fasttarget/organism/`, siempre dandole permisos al grupo para ver/editar este contenido

```bash
chown -R :dario mi_organismo_resultados/
```

## 8. Problemas frecuentes

**`conda: command not found`**

Inicializa `conda` en la shell actual:

```bash
source ~/miniconda3/etc/profile.d/conda.sh
```

**`java: command not found`**

Instala `openjdk=17` dentro del ambiente `fasttarget` y reactiva el entorno.

**`nextflow: command not found`**

Verifica que `nextflow` esté instalado y que su ubicación esté incluida en tu `PATH`.

**`Permission denied` o errores escribiendo en `/grupos/...`**

Cuidado al intentar guardar resultados dentro del repositorio compartido. Usa un directorio propio en tu `home`o verifica permisos.

**El job termina pero no encuentro la salida**

Revisa:

```bash
ls ~/fasttarget_runs
ls ~/fasttarget_runs/logs
```

Y consulta el estado del trabajo:

```bash
squeue -u $USER
sacct -j JOBID
```
