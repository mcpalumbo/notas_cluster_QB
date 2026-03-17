# Uso de LigQ_2 en el Cluster

## Descripción General

LigQ_2 es un pipeline modular de bioinformática para identificar ligandos conocidos y predichos para proteínas objetivo, partiendo de secuencias de proteínas. El repositorio se encuentra disponible en:
- **GitHub**: https://github.com/gschottlender/LigQ_2
- **Hugging Face**: https://huggingface.co/datasets/gschottlender/LigQ_2

## Ubicación en el Cluster

El pipeline se encuentra en: `/grupos/Dario/LigQ_2`

## Configuración Inicial

### 1. Crear el Ambiente Conda

Si es la primera vez que utilizas LigQ_2, debes crear el ambiente conda:

```bash
cd /grupos/Dario/LigQ_2
conda env create -f environment.yml -n ligq_2_env
```

### 2. Actualizar el Repositorio

Antes de ejecutar, actualiza el repositorio:

```bash
cd /grupos/Dario/LigQ_2
git pull
```

## Ejecución en el Cluster

### Script SLURM Básico

Crea un archivo `ligq.slurm` con el siguiente contenido:

```bash
#!/bin/bash

# Run LigQ_2 pipeline

#SBATCH --job-name="LigQ2"
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --chdir=/grupos/Dario/LigQ_2
#SBATCH --error="ligq2-%j.err"
#SBATCH --output="ligq2-%j.out"
#SBATCH --cpus-per-task=12
#SBATCH --partition=cpu

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    partición: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
echo
date +"inicio %F - %T"


# Activar ambiente conda
eval "$(conda shell.bash hook)"
conda activate ligq_2_env

# Set Variables
FASTA_FILE_PATH="/grupos/Dario/LigQ_2/organisms/mi_organismo/mi_organismo.faa"
OUTPUT_PATH="/grupos/Dario/LigQ_2/organisms/mi_organismo"

# Ejecutar LigQ
python run_ligq_2.py --input-fasta $FASTA_FILE_PATH --output-dir $OUTPUT_PATH --n-workers 12

date +"fin %F - %T"
```

### Parámetros Importantes

- **`--input-fasta`**: Ruta del archivo FASTA con las proteínas a analizar
- **`--output-dir`**: Directorio donde se guardarán los resultados
- **`--n-workers`**: Número de workers para procesamiento paralelo
- **`--cpus-per-task`**: En el script SLURM, coincide con el número de workers

### Opciones Adicionales

#### Mantener ligandos repetidos

Si deseas mantener ligandos repetidos que provengan de diferentes proteínas o tipos de búsqueda (secuencia/dominio):

```bash
python run_ligq_2.py --input-fasta $FASTA_FILE_PATH --output-dir $OUTPUT_PATH --keep-repeated-ligands --n-workers 12
```

#### Ver todas las opciones disponibles

```bash
python run_ligq_2.py -h
```

## Ejecución del Job

### Enviar el Job

```bash
sbatch ligq.slurm
```

### Monitorear la Ejecución

```bash
squeue -u $USER
```

### Revisar Logs

Los archivos de salida se generan con el formato:
- **Salida estándar**: `ligq2-<JOBID>.out`
- **Errores**: `ligq2-<JOBID>.err`

## Estructura de Directorios

```
/grupos/Dario/LigQ_2/
├── run_ligq_2.py           # Script principal
├── environment.yml          # Dependencias conda
├── organisms/               # Datos de organismos
│   └── mi_organismo/
│       ├── mi_organismo.faa      # Archivo FASTA de proteínas
│       └── (resultados)
└── ligq.slurm              # Script SLURM
```

## Configuración de la Partición

El script utiliza la partición `cpu`. Para ver las particiones disponibles:

```bash
sinfo
```

## Actualización de Bases de Datos

LigQ_2 requiere actualizar periódicamente las bases de datos (PDB, ChEMBL, ZINC) y generar tablas de resultados. Para esto, usa el script `databases.slurm`.

### Script para Actualizar Bases de Datos

Crea un archivo `databases.slurm` en `/grupos/Dario/LigQ_2`:

```bash
#!/bin/bash

# Run LigQ_2 database update

#SBATCH --job-name="LigQ2_DB"
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --chdir=/grupos/Dario/LigQ_2
#SBATCH --error="ligq2_db-%j.err"
#SBATCH --output="ligq2_db-%j.out"
#SBATCH --cpus-per-task=1
#SBATCH --partition=cpu

echo "trabajo \"${SLURM_JOB_NAME}\""
echo "    id: ${SLURM_JOB_ID}"
echo "    partición: ${SLURM_JOB_PARTITION}"
echo "    nodos: ${SLURM_JOB_NODELIST}"
echo
date +"inicio %F - %T"

# Actualizar repo
git pull

# Activar ambiente conda
eval "$(conda shell.bash hook)"
conda activate ligq_2_env

# Actualizar bases de datos PDB y ChEMBL
python update_databases.py --chembl-version 36

# Actualizar base de datos ZINC
python update_zinc_databases.py

# Generar tablas de resultados (paso largo: ~3 días)
# Requiere tener generadas las bases de datos PDB, ChEMBL y ZINC
python generate_results_tables.py --regenerate

date +"fin %F - %T"
```

### Tareas de Actualización

1. **`update_databases.py --chembl-version 36`**: Actualiza las bases de datos PDB y ChEMBL (versión 36)
2. **`update_zinc_databases.py`**: Actualiza la base de datos ZINC
3. **`generate_results_tables.py --regenerate`**: Genera tablas de resultados a partir de las bases de datos actualizadas

### Consideraciones Importantes

- **Tiempo de ejecución**: El paso `generate_results_tables.py` es muy largo (~3 días)
- **Requisitos**: Las bases de datos PDB, ChEMBL y ZINC deben ser generadas antes de ejecutar `generate_results_tables.py`
- **CPUs**: Este script usa solo 1 CPU, ya que las actualizaciones de bases de datos no se paralelizan
- **Logs**: Se generan archivos `ligq2_db-<JOBID>.out` y `ligq2_db-<JOBID>.err`

### Enviar Job de Actualización

```bash
sbatch databases.slurm
```

## Notas Importantes

- El ambiente conda debe crearse solo una vez
- La actualización del repositorio es recomendada antes de cada ejecución
- El número de workers debe ser menor o igual a los CPUs asignados
- Los resultados se guardarán en el directorio especificado en `--output-dir`
- Las actualizaciones de bases de datos son operaciones que pueden llevar tiempo.
