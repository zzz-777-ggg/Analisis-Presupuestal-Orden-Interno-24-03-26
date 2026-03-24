```
# BLOQUE 1: EXTRACCIÓN Y GRAFICACIÓN DE DATOS PRESUPUESTALES DE CONSULTA AMIGABLE
# INSTALACION DE PAQUETES Y DEPENDENCIAS
%pip install pandas
%pip install lxml

import pandas as pd
url = "https://apps5.mineco.gob.pe/transparencia/Navegador/Navegar_7.aspx?0=&30=0030&21=&y=2025"  # URL asociada a la información de interés
ls = pd.read_html(url, encoding="utf-8") # Lee el enlace y guarda la información asociada
print(f"Objeto cargado usando: {url}")

## BLOQUE MARKDOWN

**INDEXACION**


### Cuadro 2: Correspondencia de columnas del Dataframe con columnas de CA
| Nº | Nombre/Concepto                     |
|----|-------------------------------------|
| 1  | Nombre/Concepto                     |
| 2  | PIA (Presupuesto Inicial de Apertura)       |
| 3  | PIM (Presupuesto Institucional Modificado) |
| 4  | Certificación                       |
| 5  | Compromiso Anual                    |
| 6  | Atención de Compromiso Mensual     |
| 7  | Devengado                          |
| 8  | Girado                             |
| 9  | Avance (%)                         |

* Tip 1: Para encontrar el número de tabla debe examinarse visualmente la lista. Sin embargo, en la mayoría de casos, la tabla con la información principal normalmente se obtiene indexando la lista con 7, 8 o 9. Esto depende del nivel de profundidad de la búsqueda en CA.

## FIN DE BLOQUE MARKDOWN


## DISPLAY DE LOS ULTIMOS ELEMENTOS DE LA LISTA
df = ls[9] # Selecciona la tabla 9 de la lista
display(df.tail())


## SELECCION DE PIA, PIM, DEVENGADO Y AVANCE
df = df.iloc[:,[1,2,3,7,9]] # Selecciona el Pliego y el Devengado
df.columns = ["Pliego","PIA","PIM", "Devengado","Avance"] # Renombra las columnas seleccionadas
display(df.tail())

## BLOQUE MARKDOWN
### **Extraccion de datos anuales  (PANEL DE DATOS REGIONALES)** 
## FIN DE BLOQUE MARKDOWN


%%time
import pandas as pd
import os  # Librería para manejar rutas del sistema

# Haciendo bucle
bd_oi = pd.DataFrame()

for i in range(2015, 2026):
    url = f"https://apps5.mineco.gob.pe/transparencia/Navegador/Navegar_7.aspx?0=&30=0030&21=&y={i}"
    # Lee el enlace y selecciona las columnas de interés
    df = pd.read_html(url, encoding="utf-8")[9].iloc[:,[1,3,7,9]] 
    # Renombra las columnas
    df.columns = ["Región", "Presupuesto", "Devengado", "Ejecución"] 
    df["Año"] = i
    print(f"Descarga de: {url}")
    bd_oi = pd.concat([bd_oi, df])

bd_oi.reset_index(drop=True, inplace=True)

# --- CONFIGURACIÓN DE DESCARGA ---
nombre_archivo = "reporte_ejecucion_mef_2015_2025.xlsx"
bd_oi.to_excel(nombre_archivo, index=False)

# Línea que indica la ruta exacta de descarga
ruta_completa = os.path.abspath(nombre_archivo)
print(f"\n✅ ¡Éxito! El archivo se ha descargado en: {ruta_completa}")
# ---------------------------------

display(bd_oi)
print("Tiempo de ejecución:")

# BLOQUE MARKDOWN
## **Gráficos**
# FIN DE BLOQUE MARKDOWN

import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
import matplotlib as mpl
from scipy.interpolate import make_interp_spline
import numpy as np
import os

# 1. Preparar datos y ruta ================================================
ruta_excel = r"C:\Users\edwar\OneDrive\Documents\reporte_ejecucion_mef_2015_2025.xlsx"

if os.path.exists(ruta_excel):
    # Cargamos el archivo Excel
    df_full = pd.read_excel(ruta_excel)
else:
    print(f"Error: No se encontró el archivo en {ruta_excel}")
    exit()

# Nombres exactos y configuración de estilos
regiones_config = {
    "CALLAO": {"nombre": "07: PROVINCIA CONSTITUCIONAL DEL CALLAO", "color": "#C00000", "estilo": "suave"},
    "MADRE DE DIOS": {"nombre": "17: MADRE DE DIOS", "color": "#2E75B6", "estilo": "barra"},
    "TUMBES": {"nombre": "24: TUMBES", "color": "#548235", "estilo": "puntos"}
}

# Configuración estética
mpl.rcParams['font.family'] = 'Arial'
fig, ax = plt.subplots(figsize=(12, 7))

# Estilo de ejes minimalista
for s in ["top", "right"]:
    ax.spines[s].set_visible(False)
for s in ["left", "bottom"]:
    ax.spines[s].set_color("#808080")
    ax.spines[s].set_linewidth(0.64)
ax.tick_params(axis="both", width=0.64, labelsize=11)

# 2. Graficar cada región con su estilo específico =========================
for label, config in regiones_config.items():
    # Filtrar, limpiar nulos y ordenar por año
    df_reg = df_full[df_full["Región"] == config["nombre"]].dropna(subset=["Ejecución"]).sort_values("Año")
    
    x = df_reg["Año"].values
    y = df_reg["Ejecución"].values

    if config["estilo"] == "suave":
        # Curva suavizada (Spline)
        x_suave = np.linspace(x.min(), x.max(), 300)
        y_suave = make_interp_spline(x, y, k=3)(x_suave)
        ax.plot(x_suave, y_suave, color=config["color"], lw=3, label=f"{label} (Línea)", zorder=3)
    
    elif config["estilo"] == "barra":
        # Gráfico de barras con transparencia
        ax.bar(x, y, color=config["color"], alpha=0.3, width=0.6, label=f"{label} (Barras)", zorder=1)
    
    elif config["estilo"] == "puntos":
        # Solo puntos destacados
        ax.scatter(x, y, color=config["color"], s=100, label=f"{label} (Puntos)", edgecolors='white', zorder=4)
        # Línea punteada muy fina para conectar los puntos
        ax.plot(x, y, color=config["color"], lw=1, linestyle='--', alpha=0.5, zorder=2)

    # Etiquetas de datos (Porcentajes) - Corregido el comentario faltante
    for xi, yi in zip(x, y):
        ax.text(xi, yi + 0.5, f"{yi:.1f}%", ha="center", va="bottom", 
                fontsize=9, color=config["color"], fontweight='bold')

# 3. Reescala del eje Y y ajustes finales =================================
ax.set_ylim(75, 100) # Subimos un poco el tope para que las etiquetas no se corten
ax.set_xticks(df_full["Año"].unique()) 

plt.title("Ejecución Presupuestal en Seguridad Ciudadana y Orden Interno \n de las regiones más afectadas por la delincuencia (anual en %, 2015-2025)", 
          fontsize=15, fontweight='bold', pad=25)
ax.set_ylabel("Porcentaje de Ejecución (%)", fontsize=11, labelpad=10)
ax.set_xlabel("Año Fiscal", fontsize=11, labelpad=10)

# Leyenda
ax.legend(fontsize=10, frameon=False, loc='upper center', 
          bbox_to_anchor=(0.5, -0.12), ncol=3)

plt.tight_layout()

# Guardar imagen
output_img = os.path.join(os.path.dirname(ruta_excel), "evolucion_ejecucion_multiformato.png")
plt.savefig(output_img, dpi=300, bbox_inches='tight')

print(f"Gráfico generado exitosamente en: {output_img}")
plt.show()

# BLOQUE 2: GENERACION DE GRÁFICOS COROPLÉTICOS DE ESTADO DE SITUACIÓN
## Generación de mapas coropléticos para análisis presupuestal del PP 0030
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
from matplotlib.colors import LinearSegmentedColormap
import os

# 1. Ruta local del archivo
ruta_excel = r"C:\Users\edwar\OneDrive\Documents\Datos delitos pr_25_26.xlsx"

# 2. Carga y limpieza de datos
if os.path.exists(ruta_excel):
    # Leemos la Hoja2 del excel
    df = pd.read_excel(ruta_excel, sheet_name='Hoja2')
    
# Limpieza de nombres de columnas
df.columns = df.columns.astype(str).str.strip().str.lower()
    
# --- TRANSFORMACIÓN DE DATOS ---
# Convertimos decimales a porcentajes
df['victimizacion_pct'] = df['victimizacion'] * 100
df['ejecucion_pct'] = df['ejecucion_25'] * 100
    
# Escalamos presupuesto a millones (S/ 1,000,000)
df['presupuesto_mllns'] = df['presupuesto_26'] / 1_000_000
if 'departamento' not in df.columns:
        print(f"Error: No encontré la columna 'departamento'. Columnas: {list(df.columns)}")
        exit()
else:
    print("Error: No se encontró el archivo en la ruta.")
    exit()

# 3. Cargar Geometría de Departamentos
geojson_url = "https://raw.githubusercontent.com/juaneladio/peru-geojson/master/peru_departamental_simple.geojson"
gdf = gpd.read_file(geojson_url)

# 4. Normalizar nombres para el cruce
def normalize(s):
    import unicodedata
    if pd.isna(s): return ""
    s = str(s).upper().replace('REGION ', '').strip()
    return ''.join(c for c in unicodedata.normalize('NFD', s) if unicodedata.category(c) != 'Mn')

gdf['NOMBDEP_NORM'] = gdf['NOMBDEP'].apply(normalize)
df['DEPT_NORM'] = df['departamento'].apply(normalize)

merged = gdf.merge(df, left_on='NOMBDEP_NORM', right_on='DEPT_NORM', how='left')

# 5. Escala de Color (Amarillo -> Rojo Oscuro)
colors = ["#FFFFCC", "#FFD700", "#FF4500", "#800000"]
cmap_custom = LinearSegmentedColormap.from_list("seguridad_peru", colors)

# 6. Generar bloque de 4 mapas (2x2)
fig, axes = plt.subplots(2, 2, figsize=(18, 24))
fig.subplots_adjust(hspace=0.1, wspace=0.1)

# Definimos las columnas transformadas y sus títulos
maps_to_plot = [
    ('victimizacion_pct', 'Tasa de Victimización (en %)', axes[0, 0]),
    ('tasa_homicidios', 'Tasa de Homicidios (x100k hab)', axes[0, 1]),
    ('presupuesto_mllns', 'Presupuesto 2026 (en Millones S/)', axes[1, 0]),
    ('ejecucion_pct', 'Ejecución Presupuestal 2025 (en %)', axes[1, 1])
]

for col, title, ax in maps_to_plot:
    if col in merged.columns:
        merged.plot(
            column=col, 
            cmap=cmap_custom, 
            linewidth=0.8, 
            ax=ax, 
            edgecolor='black', 
            legend=True,
            legend_kwds={'shrink': 0.5, 'orientation': 'horizontal', 'pad': 0.01}
        )
        
# Resaltar Callao
callao = merged[merged['NOMBDEP_NORM'] == 'CALLAO']
        if not callao.empty:
            callao.plot(ax=ax, facecolor='none', edgecolor='black', linewidth=1.5)

# --- AÑADIR ETIQUETAS AL MAPA DE PRESUPUESTO ---
if col == 'presupuesto_mllns':
            for idx, row in merged.iterrows():
                if pd.notnull(row[col]):
                    # Obtener el centro del departamento para poner el texto
                    centro = row.geometry.centroid
                    ax.annotate(text=f"{row[col]:.1f}M", 
                                xy=(centro.x, centro.y),
                                horizontalalignment='center',
                                fontsize=8, 
                                fontweight='bold',
                                color='black',
                                # Pequeña sombra blanca para legibilidad
                                path_effects=None) 
    else:
        ax.text(0.5, 0.5, f'Columna "{col}" no encontrada', ha='center')
    ax.set_title(title, fontsize=16, fontweight='bold', pad=10)
    ax.axis('off')

plt.suptitle('Perspectiva presupuestal de seguridad ciudadana en Perú (2025-2026)', 
             fontsize=24, y=0.96, fontweight='bold')


# Guardar resultado
output_path = os.path.join(os.path.dirname(ruta_excel), "analisis_seguridad_final.png")
plt.savefig(output_path, dpi=300, bbox_inches='tight')
print(f"Proceso terminado. Imagen guardada en: {output_path}")
plt.show()
