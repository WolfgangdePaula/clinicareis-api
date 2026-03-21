"""
Proxy API - Medicoagora
Substitui o container 'curso_n8n_clinicareis-api:5000'
Endpoints compatíveis com o workflow n8n existente.

Instalação:
  pip install fastapi uvicorn requests beautifulsoup4 lxml

Execução:
  python medicoagora_proxy.py

O servidor roda em http://localhost:5000
Se o n8n rodar em Docker, altere as URLs dos tools para:
  http://host.docker.internal:5000/clinicareis/...
"""

import json
import re

import requests
from bs4 import BeautifulSoup
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

app = FastAPI(title="Medicoagora Proxy")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# ─── Configuração ────────────────────────────────────────────────────────────
BASE_URL   = "https://medicoagora.com.br"
WS_PATH    = f"{BASE_URL}/AgndOnline.aspx"     # wsPath real descoberto no HTML
NS         = "26100949971"                      # CPF da agenda (da URL do medicoagora)
FORM_URL   = f"{BASE_URL}/agndonline.aspx?ns={NS}&tema=padrao"

# Códigos de especialidade (valor real do <select> cbxEspecialidade na página)
ESPECIALIDADE_MAP = {
    "anestesiologista":   "225151_10",
    "cirurgiao plastico": "225235_11",
    "cirurgia plastica":  "225235_11",
    "proctologista":      "225280_12",
    "proctologia":        "225280_12",
}

# Códigos de convênio (valor real do <select> cbxConvenio na página)
CONVENIO_MAP = {
    "particular":  "99999",
    "bradesco":    "5711",
    "mediservice": "333689",
}

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Referer": FORM_URL,
    "Origin": BASE_URL,
}

# Sessão compartilhada (mantém cookies entre chamadas)
session = requests.Session()
session.headers.update(HEADERS)


# ─── Modelos de Entrada ───────────────────────────────────────────────────────

class BuscarRequest(BaseModel):
    convenio: str
    especialidade: str | None = None   # opcional: filtra por especialidade
    nome_medico: str | None = None     # opcional: filtra pelo nome do profissional
    agenda: str | None = None          # opcional: filtra por agenda específica

class ReservarRequest(BaseModel):
    agenda: str
    data: str       # YYYYMMDD
    hora: str       # HH:MM
    ns: str | None = None

class AgendarRequest(BaseModel):
    codReserva: str | None = None
    agenda: str
    data: str
    hora: str
    paciNome: str
    paciCPF: str
    paciFoneCel: str
    paciEmail: str
    paciDtNasc: str
    codConvenio: str
    codTipoAgendmnto: str
    nomeMedico: str | None = None
    paciFone2: str | None = ""

class CancelarRequest(BaseModel):
    codReserva: str
    agenda: str
    data: str
    hora: str
    ns: str | None = None


# ─── Helpers ─────────────────────────────────────────────────────────────────

def get_viewstate(agenda: str | None = None) -> tuple[dict, str]:
    """Faz GET na página para obter o ViewState. Retorna (campos, url_usada)."""
    url = FORM_URL
    if agenda:
        url = f"{url}&agenda={agenda}"
    resp = session.get(url, timeout=15)
    resp.raise_for_status()
    soup = BeautifulSoup(resp.text, "lxml")
    fields = {
        "__VIEWSTATE":          soup.find("input", {"name": "__VIEWSTATE"})["value"],
        "__VIEWSTATEGENERATOR": soup.find("input", {"name": "__VIEWSTATEGENERATOR"})["value"],
        "__EVENTVALIDATION":    soup.find("input", {"name": "__EVENTVALIDATION"})["value"],
    }
    return fields, url


def parse_slots(html: str, limit: int = 5) -> list[dict]:
    """
    Extrai horários da página de resultados.
    Estrutura real descoberta:
      <table class="table-horarios" data-dia="20260319" data-agenda="SILVANA">
        <tr><td><span class="btHora" data-hora="14:00">14:00</span></td></tr>
    """
    soup = BeautifulSoup(html, "lxml")
    slots = []

    for tbl in soup.select("table.table-horarios"):
        data_dia   = tbl.get("data-dia", "")    # YYYYMMDD
        data_agenda = tbl.get("data-agenda", NS)  # nome da agenda / médico

        for span in tbl.select("span.btHora[data-hora]"):
            hora = span.get("data-hora", "")
            if data_dia and hora:
                slots.append({
                    "agenda": data_agenda,
                    "data":   data_dia,
                    "hora":   hora,
                    "medico": data_agenda,   # agenda = nome do médico neste sistema
                })
                if len(slots) >= limit:
                    return slots

    return slots


def ws_call(method_name: str, payload: dict) -> dict:
    """
    Chama um WebMethod do AgndOnline.aspx.
    Formato: POST JSON com Content-Type: application/json; charset=utf-8
    Resposta: {"d": "JSON_STRING"} — padrão ASP.NET ScriptMethod
    """
    url = f"{WS_PATH}/{method_name}"
    payload.setdefault("ns", NS)

    headers = {
        "Content-Type": "application/json; charset=utf-8",
        "Accept": "application/json, text/javascript, */*; q=0.01",
        "X-Requested-With": "XMLHttpRequest",
    }

    try:
        resp = session.post(url, data=json.dumps(payload), headers=headers, timeout=15)
        resp.raise_for_status()
        outer = resp.json()
        inner = outer.get("d", "{}")
        return json.loads(inner) if isinstance(inner, str) else inner
    except requests.HTTPError as e:
        raise HTTPException(status_code=502, detail=f"Medicoagora HTTP {e.response.status_code}: {e}")
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"Erro medicoagora: {e}")


# ─── Endpoints ───────────────────────────────────────────────────────────────

@app.post("/clinicareis/buscar")
def buscar_horarios(req: BuscarRequest):
    """Busca horários disponíveis via form POST + parse HTML."""
    esp  = ESPECIALIDADE_MAP.get(req.especialidade.lower().strip(), "0") if req.especialidade else "0"
    conv = CONVENIO_MAP.get(req.convenio.lower().strip(), "99999")

    try:
        viewstate, form_url = get_viewstate(req.agenda)
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"Erro ao acessar medicoagora: {e}")

    form_data = {
        **viewstate,
        "cbxEspecialidade":    esp,
        "cbxConvenio":         conv,
        "txtNomeMedico":       req.nome_medico or "",
        "txtDataMinima":       "",
        "btnBuscarProfissionais": "Buscar",
    }

    try:
        resp = session.post(form_url, data=form_data, timeout=20)
        # Não usa raise_for_status() — medicoagora pode retornar 500
        # mas ainda assim entregar o HTML com os resultados
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"Erro ao buscar horários: {e}")

    slots = parse_slots(resp.text, limit=5)

    if not slots:
        return {
            "ok": False,
            "mensagem": "Nenhum horário disponível para os critérios informados.",
            "horarios": [],
        }

    return {
        "ok": True,
        "horarios": slots,
        "mensagem": f"{len(slots)} horário(s) encontrado(s).",
    }


@app.post("/clinicareis/reservar")
def reservar_horario(req: ReservarRequest):
    """Pré-reserva um horário (válido por ~10 minutos)."""
    resultado = ws_call("wsPostReservaHorario", {
        "ns":     req.ns or NS,
        "agenda": req.agenda,
        "data":   req.data,
        "hora":   req.hora,
    })

    status = resultado.get("StatusCod", -1)
    msg    = resultado.get("Mensagem", "")

    if status == 0:
        return {
            "ok":         True,
            "codReserva": msg,
            "mensagem":   f"Horário reservado. Código: {msg}",
        }
    return {
        "ok":       False,
        "mensagem": f"Falha ao reservar: {msg} (código {status})",
    }


@app.post("/clinicareis/agendar")
def confirmar_agendamento(req: AgendarRequest):
    """Confirma o agendamento definitivo com todos os dados do paciente."""
    # Pré-reserva automática se codReserva não foi informado
    cod_reserva = req.codReserva or ""
    if not cod_reserva:
        reserva = ws_call("wsPostReservaHorario", {
            "ns":     NS,
            "agenda": req.agenda,
            "data":   req.data,
            "hora":   req.hora,
        })
        if reserva.get("StatusCod", -1) != 0:
            return {"ok": False, "mensagem": f"Falha ao pré-reservar horário: {reserva.get('Mensagem', '')}"}
        cod_reserva = reserva.get("Mensagem", "")

    resultado = ws_call("wsPostEfetivaAgendamento", {
        "ns":               NS,
        "agenda":           req.agenda,
        "data":             req.data,
        "hora":             req.hora,
        "paciCPF":          req.paciCPF,
        "paciNome":         req.paciNome,
        "paciFoneCel":      req.paciFoneCel,
        "paciFone2":        req.paciFone2 or "",
        "paciEmail":        req.paciEmail,
        "paciDtNasc":       req.paciDtNasc,
        "codConvenio":      req.codConvenio,
        "codReserva":       cod_reserva,
        "clinicaNome":      "Clínica Reis",
        "clinicaEnd":       "Av. Constantino Nery, 3010, Manaus - AM",
        "clinicaFones":     "(92) 98160-7756",
        "nomeMedico":       req.nomeMedico or "",
        "codTipoAgendmnto": req.codTipoAgendmnto,
    })

    status = resultado.get("StatusCod", -1)
    msg    = resultado.get("Mensagem", "")

    if status == 0:
        return {"ok": True,  "mensagem": f"Agendamento confirmado! {msg}"}
    return {"ok": False, "mensagem": f"Falha ao confirmar: {msg} (código {status})"}


@app.post("/clinicareis/cancelar")
def cancelar_agendamento(req: CancelarRequest):
    """Cancela um agendamento existente."""
    resultado = ws_call("wsPostCancelaReserva", {
        "ns":         req.ns or NS,
        "agenda":     req.agenda,
        "data":       req.data,
        "hora":       req.hora,
        "codReserva": req.codReserva,
    })

    status = resultado.get("StatusCod", -1)
    msg    = resultado.get("Mensagem", "")

    if status == 0:
        return {"ok": True,  "mensagem": f"Cancelado. {msg}"}
    return {"ok": False, "mensagem": f"Falha ao cancelar: {msg} (código {status})"}


@app.post("/debug/buscar")
def debug_buscar(req: BuscarRequest):
    """Debug: retorna HTML bruto e parâmetros usados para inspecionar o scraping."""
    esp  = ESPECIALIDADE_MAP.get(req.especialidade.lower().strip(), "0") if req.especialidade else "0"
    conv = CONVENIO_MAP.get(req.convenio.lower().strip(), "99999")

    viewstate, form_url = get_viewstate(req.agenda)

    form_data = {
        **viewstate,
        "cbxEspecialidade":    esp,
        "cbxConvenio":         conv,
        "txtNomeMedico":       req.nome_medico or "",
        "txtDataMinima":       "",
        "btnBuscarProfissionais": "Buscar",
    }

    resp = session.post(form_url, data=form_data, timeout=20)

    soup = BeautifulSoup(resp.text, "lxml")
    tabelas = [str(t) for t in soup.select("table.table-horarios")]

    return {
        "esp_code": esp,
        "conv_code": conv,
        "form_url": form_url,
        "status_code": resp.status_code,
        "tabelas_encontradas": len(tabelas),
        "tabelas_html": tabelas,
        "html_snippet": resp.text[5000:8000],
    }


@app.get("/health")
def health():
    return {"status": "ok", "ns": NS, "ws_path": WS_PATH}


# ─── Main ─────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5000)
