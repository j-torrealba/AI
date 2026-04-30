## Análisis Semanal de Licitaciones — Fundación Invictus



**Objetivo:** Descargar las licitaciones activas desde la API de Mercado Público Chile, filtrar las relevantes para Fundación Invictus (rubros: Batas y Overoles, Uniformes, Ropa de Trabajo, Muebles de Oficina, Muebles Independientes), y generar un reporte Excel profesional con análisis y enlaces directos.



---



### PASO 1 — Ejecuta el siguiente script Python con la herramienta Bash:



```python

import json, ssl, time, urllib.request, urllib.parse

from datetime import datetime

from pathlib import Path

from openpyxl import Workbook

from openpyxl.styles import Alignment, Border, Font, PatternFill, Side

from openpyxl.utils import get_column_letter



# ── CONFIGURACIÓN ─────────────────────────────────────────────────────────

MP_API_KEY = "BA4DCB3A-8C8E-45C9-84CA-B463A84C135A"

EMAILS_DESTINO = ["jtorrealba@fundacioninvictus.cl", "andrea@fundacioninvictus.cl"]



SEARCH_TERMS = ["uniformes","ropa de trabajo","overoles","batas","vestuario","muebles","mobiliario"]



CATEGORIES = {

    "Batas y Overoles":    ["bata","overol","plana quirúrgica","plana quirurgica","pijama","scrub","mandil"],

    "Uniformes":           ["uniforme","uniformes clínicos","uniformes para funcionarios","uniformes para trabajadoras","tenida"],

    "Ropa de Trabajo":     ["ropa de trabajo","vestuario institucional","vestuario laboral","indumentaria","prendas de vestir","prendas diversas","epp"],

    "Muebles / Mobiliario":["mueble","mobiliario","gavetero","escritorio","silla de oficina","muebles de oficina"],

}



FALSE_POSITIVES = [

    "seguro de incendio","servicio de vigilancia","servicio de aseo","limpieza industrial",

    "servicio odontológico","prótesis dental","control de plagas","demolición","mensura",

    "multicancha","reposición plaza","mejoramiento plaza","mejoramiento parque",

    "acera peatonal","cubierta de gimnasio","mantenimiento eléctrico","reparación de sellos",

    "ferretería","envases de colación","arriendo de inmueble","obras de emergencia",

    "desfibrilador","corbata",

]



# ── API ────────────────────────────────────────────────────────────────────

MP_BASE = "https://api.mercadopublico.cl/servicios/v1/publico/licitaciones.json"



def call_api(term, cantidad=1000):

    params = f"?estado=publicada&nombre={urllib.parse.quote(term)}&cantidad={cantidad}&ticket={MP_API_KEY}"

    ctx = ssl._create_unverified_context()

    try:

        with urllib.request.urlopen(MP_BASE+params, context=ctx, timeout=20) as r:

            return json.load(r).get("Listado") or []

    except Exception as e:

        print(f"  ⚠ Error '{term}': {e}"); return []



def fetch_all():

    seen, items = set(), []

    for term in SEARCH_TERMS:

        print(f"  🔍 '{term}'...")

        for it in call_api(term):

            lid = it.get("CodigoExterno","")

            if lid and lid not in seen:

                seen.add(lid); items.append(it)

        time.sleep(0.3)

    print(f"  → {len(items)} únicos descargados")

    return items



def normalize(it):

    org = it.get("Organismo",{})

    m = it.get("MontoEstimado") or it.get("Monto") or 0

    return {

        "id":      it.get("CodigoExterno",""),

        "nombre":  it.get("Nombre","").strip(),

        "org":     org.get("NombreOrganismo","").strip() if isinstance(org,dict) else str(org),

        "tipo":    it.get("Tipo",""),

        "fecha":   (it.get("FechaPublicacion","") or "")[:10],

        "cierre":  (it.get("FechaCierre","") or "")[:10],

        "monto":   float(m) if m else 0,

        "desc":    it.get("Descripcion","").strip()[:250],

    }



def categorize(n):

    text = (n["nombre"]+" "+n["desc"]).lower()

    cats = [c for c,kws in CATEGORIES.items() if any(k in text for k in kws)]

    if not cats: return None, None

    nom = n["nombre"].lower()

    principal = ["uniforme","ropa de trabajo","bata","overol","mueble","mobiliario","vestuario","indumentaria","prenda"]

    for fp in FALSE_POSITIVES:

        if fp in nom and not any(p in nom for p in principal):

            return None, None

    pri = "Alta" if n["monto"] > 15_000_000 else ("Media" if n["monto"] > 2_000_000 or n["monto"]==0 else "Baja")

    return ", ".join(cats), pri



# ── Excel ──────────────────────────────────────────────────────────────────

C_DB="1F3864"; C_MB="2E75B6"; C_LB="D6E4F0"; C_W="FFFFFF"; C_G="F2F2F2"

PF={"Alta":"FFE699","Media":"DDEBF7","Baja":C_G}

PFC={"Alta":"7F6000","Media":"1F3864","Baja":"595959"}

def _f(c): return PatternFill("solid",fgColor=c)

def _b():

    s=Side(style="thin",color="BFBFBF")

    return Border(left=s,right=s,top=s,bottom=s)

def _c(): return Alignment(horizontal="center",vertical="center",wrap_text=True)

def _l(): return Alignment(horizontal="left",vertical="center",wrap_text=True)

def _hf(sz=10): return Font(name="Arial",size=sz,bold=True,color=C_W)

def _bf(sz=9):  return Font(name="Arial",size=sz)



def build_excel(lics, path, fecha):

    wb = Workbook()

    ws = wb.active; ws.title="Resumen Ejecutivo"; ws.sheet_view.showGridLines=False

    ws.merge_cells("A1:J1")

    ws["A1"]=f"ANÁLISIS SEMANAL DE LICITACIONES — FUNDACIÓN INVICTUS  ({fecha})"

    ws["A1"].font=Font(name="Arial",size=14,bold=True,color=C_W); ws["A1"].fill=_f(C_DB); ws["A1"].alignment=_c()

    ws.row_dimensions[1].height=34

    ws.merge_cells("A2:J2")

    ws["A2"]="Rubros: Batas y Overoles · Uniformes · Ropa de Trabajo · Muebles de Oficina · Muebles Independientes"

    ws["A2"].font=Font(name="Arial",size=10,italic=True,color=C_W); ws["A2"].fill=_f(C_MB); ws["A2"].alignment=_c()

    ws.row_dimensions[2].height=18

    ws.merge_cells("A3:J3")

    alta=sum(1 for l in lics if l["pri"]=="Alta")

    ws["A3"]=f"Fuente: API Mercado Público  |  Semana: {fecha}  |  Total relevantes: {len(lics)}  |  Alta prioridad: {alta}"

    ws["A3"].font=Font(name="Arial",size=9,italic=True,color="595959"); ws["A3"].fill=_f(C_G); ws["A3"].alignment=_c()

    ws.row_dimensions[3].height=14; ws.row_dimensions[4].height=6

    hdrs=["#","ID Licitación","Nombre","Organismo","Tipo","Fecha Pub.","Fecha Cierre","Monto Est. ($)","Categoría","Prioridad","Acceso"]

    ws_=[4,20,42,32,6,12,12,18,26,10,22]

    for ci,(h,w) in enumerate(zip(hdrs,ws_),1):

        c=ws.cell(row=5,column=ci,value=h); c.font=_hf(9); c.fill=_f(C_DB); c.alignment=_c(); c.border=_b()

        ws.column_dimensions[get_column_letter(ci)].width=w

    ws.row_dimensions[5].height=26; ws.freeze_panes="A6"

    def murl(lid): return f"https://www.mercadopublico.cl/Procurement/Modules/RFB/DetailsAcquisition.aspx?qs=OE3bYinNbq0kNZvpMnEkDA==&idlicitacion={lid}"

    for i,l in enumerate(lics):

        bg=C_W if i%2==0 else "EEF4FB"

        mf=f"${l['monto']:,.0f}" if l['monto'] else "No publicado"

        row=6+i

        for ci,v in enumerate([i+1,l['id'],l['nombre'],l['org'],l['tipo'],l['fecha'],l['cierre'],mf,l['cats'],l['pri']],1):

            c=ws.cell(row=row,column=ci,value=v); c.border=_b()

            if ci==10: c.font=Font(name="Arial",size=9,bold=True,color=PFC[l['pri']]); c.fill=_f(PF[l['pri']]); c.alignment=_c()

            elif ci==8: c.font=Font(name="Arial",size=9,bold=True,color="375623"); c.fill=_f(bg); c.alignment=_l()

            elif ci in[1,5]: c.font=_bf(); c.fill=_f(bg); c.alignment=_c()

            else: c.font=_bf(); c.fill=_f(bg); c.alignment=_l()

        lc=ws.cell(row=row,column=11,value="Ver →")

        lc.hyperlink=murl(l['id']); lc.font=Font(name="Arial",size=9,color="0563C1",underline="single")

        lc.fill=_f(bg); lc.border=_b(); lc.alignment=_c()

        ws.row_dimensions[row].height=36



    # Hoja Top 15

    ws2=wb.create_sheet("Top 15"); ws2.sheet_view.showGridLines=False

    ws2.merge_cells("A1:H1"); ws2["A1"]=f"TOP OPORTUNIDADES — {fecha}"

    ws2["A1"].font=Font(name="Arial",size=13,bold=True,color=C_W); ws2["A1"].fill=_f(C_DB); ws2["A1"].alignment=_c()

    ws2.row_dimensions[1].height=30; ws2.row_dimensions[2].height=6

    th=["#","ID","Organismo","Tipo","Monto Est.","Fecha Cierre","Categoría","Prioridad"]

    tw=[6,22,38,6,20,12,28,10]

    for ci,(h,w) in enumerate(zip(th,tw),1):

        c=ws2.cell(row=3,column=ci,value=h); c.font=_hf(9); c.fill=_f(C_DB); c.alignment=_c(); c.border=_b()

        ws2.column_dimensions[get_column_letter(ci)].width=w

    ws2.row_dimensions[3].height=24; ws2.freeze_panes="A4"

    medals=["1°","2°","3°"]+[f"{n}°" for n in range(4,16)]

    for i,l in enumerate(lics[:15]):

        bg=C_W if i%2==0 else "EDF2F9"; row=4+i

        mf=f"${l['monto']:,.0f}" if l['monto'] else "No publicado"

        for ci,v in enumerate([medals[i],l['id'],l['org'],l['tipo'],mf,l['cierre'],l['cats'],l['pri']],1):

            c=ws2.cell(row=row,column=ci,value=v); c.border=_b()

            if ci==1: c.font=Font(name="Arial",size=11,bold=True,color=C_DB); c.fill=_f("FFF2CC"); c.alignment=_c()

            elif ci==8: c.font=Font(name="Arial",size=9,bold=True,color=PFC[l['pri']]); c.fill=_f(PF[l['pri']]); c.alignment=_c()

            elif ci==4: c.font=_bf(); c.fill=_f(bg); c.alignment=_c()

            else: c.font=_bf(); c.fill=_f(bg); c.alignment=_l()

        ws2.row_dimensions[row].height=36



    wb.save(path)



# ── MAIN ───────────────────────────────────────────────────────────────────

print("="*55)

print("  ANÁLISIS SEMANAL — FUNDACIÓN INVICTUS")

print("="*55)

raw = fetch_all()

results = []

for it in raw:

    n = normalize(it)

    cats, pri = categorize(n)

    if cats:

        n["cats"] = cats; n["pri"] = pri

        results.append(n)



order = {"Alta":0,"Media":1,"Baja":2}

results.sort(key=lambda x:(order.get(x["pri"],3),-(x["monto"] or 0)))



fecha = datetime.now().strftime("%Y-%m-%d")

out = f"/tmp/Licitaciones_Invictus_{fecha}.xlsx"

build_excel(results, out, fecha)

print(f"\n✅ Excel generado: {out}")

print(f"   Total: {len(results)} | Alta: {sum(1 for r in results if r['pri']=='Alta')}")



print("\n📋 TOP 5 OPORTUNIDADES:")

for i,l in enumerate(results[:5],1):

    monto_str = f"${l['monto']:,.0f}" if l['monto'] else "Monto no publicado"

    print(f"  {i}. [{l['pri']}] {l['nombre'][:60]} | {l['org'][:35]} | {monto_str}")



import json

summary = {

    "fecha": fecha,

    "total": len(results),

    "alta": sum(1 for r in results if r['pri']=='Alta'),

    "media": sum(1 for r in results if r['pri']=='Media'),

    "baja": sum(1 for r in results if r['pri']=='Baja'),

    "top5": results[:5],

    "excel_path": out

}

with open("/tmp/invictus_summary.json","w") as f:

    json.dump(summary, f, ensure_ascii=False)

```



### PASO 2 — Después de ejecutar el script:



1. Copia el archivo Excel generado en `/tmp/Licitaciones_Invictus_FECHA.xlsx` al workspace: `cp /tmp/Licitaciones_Invictus_*.xlsx /sessions/*/mnt/outputs/ 2>/dev/null || true`



2. Envía el reporte por Gmail a **jtorrealba@fundacioninvictus.cl** y **andrea@fundacioninvictus.cl** con:

   - Asunto: `[Invictus] Análisis Licitaciones Semana del {FECHA} — {N} oportunidades ({ALTA} alta prioridad)`

   - Cuerpo HTML profesional: incluye resumen Top 5, tabla de prioridades (Alta/Media/Baja), y explica que el Excel detallado está en la carpeta de trabajo.

   - Enviar a ambos destinatarios en el campo "to" separados por coma.



3. Si el script falla por error de red/API (EGRESS_BLOCKED o similar), crear un borrador de aviso en Gmail dirigido a **jtorrealba@fundacioninvictus.cl** y **andrea@fundacioninvictus.cl** explicando el error y que deben verificar la API Key o el acceso de red.



4. Muestra al usuario un resumen ejecutivo del análisis.



### NOTAS IMPORTANTES:

- API Key: BA4DCB3A-8C8E-45C9-84CA-B463A84C135A

- Destinatarios: jtorrealba@fundacioninvictus.cl, andrea@fundacioninvictus.cl

- Si la API de Mercado Público no responde, reportar el error y crear borrador de aviso.

- Solo reportar licitaciones en estado "publicada" (disponibles para ofertar).

- Rubros: uniformes, ropa de trabajo, overoles, batas, vestuario, muebles, mobiliario.
