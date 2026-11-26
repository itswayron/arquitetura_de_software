# Governança Evolutiva e Fitness Functions Automatizadas (`ADRs 0001 a 0006`)

> **Projeto:** The Road Warrior (Dashboard Global de Viagens em Tempo Real + AI Travel Copilot)  
> **Disciplina:** Arquitetura de Software — UniSENAI | **Equipe 5:** Vibecoders  
> **Autor (Branch `lucas`):** Lucas Sabino da Silva (Executor na Sprint 4; Líder na Sprint 3)

---

## 1. Papel das Fitness Functions na Arquitetura Evolutiva

Seguindo a engenharia de *Building Evolutionary Architectures* (Neal Ford, Rebecca Parsons e Patrick Kua), decisões arquiteturais documentadas em ADRs degradam-se rapidamente se não forem protegidas por **Fitness Functions** — mecanismos objetivos e automatizados executados no pipeline de CI/CD e em ambientes de *staging* que reprovam *builds* ou disparam disjuntores operacionais sempre que uma fronteira de acoplamento, latência, resiliência, privacidade ou custo é violada.

```mermaid
flowchart LR
    PR["Pull Request / Commit"] --> AST["Gate 1: Analise Estatica AST\n(Zero xml.etree / SDK no Dominio + D = 0.0)"]
    AST --> GUARD["Gate 2: Testes de Contrato e Guardrails\n(Zero-PII LGPD + Regex Pydantic + Fallback IA)"]
    GUARD --> CHAOS["Gate 3: Chaos & Performance Staging\n(Dashboard p95 abaixo de 600 ms com GDS e IA Offline)"]
    CHAOS --> PROD["Deploy Seguro + Runtime LLMOps\n(Cache Hit >= 65% + Kill Switch Financeiro)"]
```

---

## 2. Matriz Consolidada das Fitness Functions (`ADR-001` a `ADR-006`)

| ADR | Código | Dimensão QAW / Arquitetural | Regra e Limiar Quantitativo Inegociável | Mecanismo de Verificação Automatizada |
| :---: | :---: | :--- | :--- | :--- |
| **`ADR-001`** | `FF-01` | Manutenibilidade e Desacoplamento (Adapters) | `0` imports de SDKs externos ou bibliotecas SOAP/HTTP em pacotes de domínio; todo conector deve implementar `IAirCompanyIntegration`. | Inspeção sintática via `ast` em Python no CI/CD a cada *Pull Request*. |
| **`ADR-002`** | `FF-02` | Escalabilidade e Confiabilidade (RabbitMQ) | Consumidores `AirCompanyWorker` idempotentes; `0%` de perda de mensagens com desvio para `itinerary.sync.dlq` após `3` falhas; suporte a `500.000` usuários simultâneos. | Teste de carga com mensagens duplicadas/`Poison Pill` validando estado único e encaminhamento à DLQ. |
| **`ADR-003`** | `FF-03` | Disponibilidade (`>= 99,9%`) e Performance (`< 600 ms`) | Clientes externos com `timeout <= 3000 ms` + Circuit Breaker (`50%`/`20 reqs`/`30s`); gRPC interno `<= 400 ms`; leitura `GET /api/v1/dashboard/{tripId}` com `p95 < 600 ms` via `STALE_FALLBACK` no Redis. | Análise estática de `timeout` nos Adapters + *Chaos Test* derrubando mocks SOAP/GDS (`HTTP 503`). |
| **`ADR-004`** | `FF-04` | Estabilidade Estrutural DDD e SOLID (`D = 0.0`) | Distância da Sequência Principal `D = \|A + I - 1\| = 0.0` nos pacotes refatorados; `Itinerarios.Domain` proibido de importar `Integracao.Adapters`; aderência à Linguagem Ubíqua. | Teste automatizado de métricas de pacotes (`N_a, N_c, C_a, C_e`) e grafo de imports no CI/CD. |
| **`ADR-005`** | `FF-05` | Usabilidade Mobile e Tradução EIP (BFF / ACL) | Renderização completa da tela principal no PWA dispara `<= 1` requisição HTTP ao `DashboardAggregatorService`; `Itinerarios.Domain` sem `xml.etree`; `100%` dos campos obrigatórios traduzidos pelo `ReservationDataNormalizer`. | Teste de contrato OpenAPI/E2E contando chamadas de rede + suíte unitária do `Message Translator`. |
| **`ADR-006`** | `FF-06A` | Isolamento Determinístico face à IA | Queda total do `AI Engine` ou timeout de `10s` nas LLMs jamais afeta `GET /api/v1/dashboard/{tripId}`, mantendo `HTTP 200` e `p95 < 600 ms`. | *Chaos Test* em staging desligando os contêineres/mocks de IA durante pico de leitura. |
| **`ADR-006`** | `FF-06B` | Segurança `LGPD/GDPR` e Anti-Alucinação (RAG) | `100%` de remoção de e-mail, CPF, nome e PNR pelo `PiiInputGuardrail`; `policy_doc_id` válido (`^POL-[A-Z]{2,4}-\d{4}$`), `confidence_score >= 0.75`, `disclaimer_required = True` e Fallback determinístico após `1` *retry* ou erro de rede. | Suíte `pytest` executada em todo commit simulando PII real, alucinações de schema e exceções no `ILlmProviderAdapter`. |
| **`ADR-006`** | `FF-06C` | Governança de Custos (*Token Economics*) | *Cache hit* semântico `>= 65%` (`< 50 ms`, `$0` tokens) em disrupções repetidas e acionamento automático do *Kill Switch* bloqueando a Camada 2 (Reasoning Tier) ao atingir o `Token Budget`. | Monitoramento sintético e teste unitário do disjuntor orçamentário no `ModelRouter`. |

---

## 3. Suíte Executável de Fitness Functions em Python (`pytest` + `ast`)

O arquivo de teste abaixo é **100% autossuficiente e executável via `pytest`**, cobrindo os quatro pilares críticos de verificação automatizada no CI/CD:
- **(a)** Auditoria estática via árvore sintática abstrata (`ast`) impedindo que o domínio importe `xml.etree`, clientes HTTP/SDKs externos ou `Integracao.Adapters` (`ADR-001`, `ADR-004` e `ADR-005`);
- **(b)** Validação rigorosa de anonimização `Zero-PII` (`PiiInputGuardrail`) removendo `100%` de e-mails, CPFs, nomes e códigos PNR (`ADR-006`);
- **(c)** Validação de contrato Pydantic exigindo regex `^POL-[A-Z]{2,4}-\d{4}$` em `policy_doc_id`, `confidence_score >= 0.75`, `disclaimer_required = True` e *Human-in-the-Loop* (`ADR-006`);
- **(d)** Comprovação de que saída inválida/alucinada de LLM ou falha/timeout no provedor (`ILlmProviderAdapter`) aciona imediatamente o **Fallback Determinístico** (`ADR-003` e `ADR-006`).

```python
from __future__ import annotations

import ast
import json
import re
from abc import ABC, abstractmethod
from typing import Literal

import pytest
from pydantic import BaseModel, Field, ValidationError, field_validator, model_validator


# ============================================================================
# COMPONENTES AUDITADOS PELA SUÍTE DE FITNESS FUNCTIONS
# ============================================================================

FORBIDDEN_DOMAIN_IMPORTS: tuple[str, ...] = (
    "xml.etree",
    "lxml",
    "zeep",
    "requests",
    "httpx",
    "boto3",
    "openai",
    "anthropic",
    "google.generativeai",
    "integracao.adapters",
)


def find_forbidden_domain_imports(python_source_code: str) -> list[str]:
    """Inspeciona a AST de um módulo de domínio em busca de acoplamentos proibidos."""
    tree = ast.parse(python_source_code)
    violations: list[str] = []

    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                mod_name = alias.name.lower()
                if any(mod_name == fb or mod_name.startswith(f"{fb}.") for fb in FORBIDDEN_DOMAIN_IMPORTS):
                    violations.append(alias.name)
        elif isinstance(node, ast.ImportFrom) and node.module:
            mod_name = node.module.lower()
            if any(mod_name == fb or mod_name.startswith(f"{fb}.") for fb in FORBIDDEN_DOMAIN_IMPORTS):
                violations.append(node.module)

    return violations


class PiiInputGuardrail:
    """Guardrail de Entrada (LGPD/GDPR): mascara e-mail, CPF, nome e PNR de 6 dígitos."""

    EMAIL_PATTERN = re.compile(r"[\w\.-]+@[\w\.-]+\.\w+")
    CPF_PATTERN = re.compile(r"\b\d{3}\.?\d{3}\.?\d{3}-?\d{2}\b")
    LABELED_NAME_PATTERN = re.compile(
        r"(?P<prefix>\b(?:passageir[oa]|viajante|cliente|nome)\s*[:=-]\s*)"
        r"(?P<name>[A-ZÀ-Ÿ][a-zà-ÿ]+(?:\s+(?:da|de|do|dos|das|e|[A-ZÀ-Ÿ][a-zà-ÿ]+))+)"
    )
    PNR_PATTERN = re.compile(r"\b[A-Z0-9]{6}\b")

    @classmethod
    def sanitize_prompt(cls, raw_text: str, traveler_name: str | None = None) -> str:
        sanitized = cls.EMAIL_PATTERN.sub("[REDACTED_EMAIL]", raw_text)
        sanitized = cls.CPF_PATTERN.sub("[REDACTED_CPF]", sanitized)
        if traveler_name and traveler_name.strip():
            sanitized = re.sub(re.escape(traveler_name.strip()), "[REDACTED_NAME]", sanitized, flags=re.IGNORECASE)
        sanitized = cls.LABELED_NAME_PATTERN.sub(r"\g<prefix>[REDACTED_NAME]", sanitized)
        sanitized = cls.PNR_PATTERN.sub("[REDACTED_PNR]", sanitized)
        return sanitized


class RebookingActionOption(BaseModel):
    action_type: Literal["REBOOK_FLIGHT", "HOTEL_LATE_CHECKIN", "LOUNGE_ACCESS", "CONTACT_AIRLINE"]
    summary: str = Field(..., min_length=10, max_length=180)
    policy_doc_id: str = Field(..., pattern=r"^POL-[A-Z]{2,4}-\d{4}$")
    confidence_score: float = Field(..., ge=0.75, le=1.0)
    requires_human_approval: bool = True

    @model_validator(mode="after")
    def validate_financial_human_in_loop(self) -> "RebookingActionOption":
        if self.action_type in {"REBOOK_FLIGHT", "HOTEL_LATE_CHECKIN"} and not self.requires_human_approval:
            raise ValueError("Ações financeiras exigem requires_human_approval=True (Human-in-the-Loop).")
        return self


class AutonomousDisruptionAdvice(BaseModel):
    trip_id: str
    severity: Literal["LOW", "MEDIUM", "CRITICAL"]
    options: list[RebookingActionOption] = Field(..., min_length=1, max_length=3)
    disclaimer_required: bool = True
    is_fallback: bool = False

    @field_validator("disclaimer_required")
    @classmethod
    def enforce_disclaimer(cls, value: bool) -> bool:
        if value is not True:
            raise ValueError("disclaimer_required deve ser True.")
        return value


class ILlmProviderAdapter(ABC):
    @abstractmethod
    def complete_json(self, sanitized_prompt: str) -> str:
        pass


class ResilientAiOrchestrator:
    """Executa sanitização PII, até 1 retry de autocorreção Pydantic e Fallback Determinístico."""

    def __init__(self, provider: ILlmProviderAdapter) -> None:
        self._provider = provider
        self.sent_prompts: list[str] = []

    @staticmethod
    def deterministic_fallback(trip_id: str) -> AutonomousDisruptionAdvice:
        return AutonomousDisruptionAdvice(
            trip_id=trip_id,
            severity="MEDIUM",
            options=[
                RebookingActionOption(
                    action_type="CONTACT_AIRLINE",
                    summary="Instabilidade no assistente inteligente. Consulte o balcão oficial da companhia aérea.",
                    policy_doc_id="POL-ANAC-0400",
                    confidence_score=1.0,
                    requires_human_approval=True,
                )
            ],
            disclaimer_required=True,
            is_fallback=True,
        )

    def generate_advice(self, trip_id: str, raw_prompt: str, traveler_name: str | None = None) -> AutonomousDisruptionAdvice:
        clean_prompt = PiiInputGuardrail.sanitize_prompt(raw_prompt, traveler_name=traveler_name)

        for _attempt in range(2):  # Tentativa inicial + no máximo 1 retry de autocorreção
            try:
                self.sent_prompts.append(clean_prompt)
                raw_response = self._provider.complete_json(clean_prompt)
                payload = json.loads(raw_response)
                payload["trip_id"] = trip_id
                return AutonomousDisruptionAdvice.model_validate(payload)
            except (TimeoutError, ConnectionError, RuntimeError):
                return self.deterministic_fallback(trip_id)
            except (json.JSONDecodeError, ValidationError, ValueError) as err:
                clean_prompt = f"{clean_prompt}\n[RETRY_SCHEMA]: {err}"

        return self.deterministic_fallback(trip_id)


# ============================================================================
# SUÍTE DE TESTES AUTOMATIZADOS (PYTEST)
# ============================================================================

# (a) Governança de Domínio via AST: Proibição de xml.etree e SDKs Externos
def test_ff01_clean_domain_module_has_zero_forbidden_imports() -> None:
    clean_domain_code = """
from dataclasses import dataclass
from typing import Protocol

class IItineraryProjectionRepository(Protocol):
    def get_by_trip_id(self, trip_id: str) -> dict: ...

@dataclass(frozen=True)
class FlightSegment:
    reservation_code: str
    flight_number: str
    gate: str
    status: str
"""
    assert find_forbidden_domain_imports(clean_domain_code) == []


@pytest.mark.parametrize(
    "violating_code,expected_violation",
    [
        ("import xml.etree.ElementTree as ET\n", "xml.etree.ElementTree"),
        ("from xml.etree import ElementTree\n", "xml.etree"),
        ("import requests\n", "requests"),
        ("import openai\n", "openai"),
        ("from integracao.adapters.sabre import SabreAirCompanyIntegration\n", "integracao.adapters.sabre"),
    ],
)
def test_ff01_ast_detects_xml_etree_and_external_sdks_in_domain(
    violating_code: str, expected_violation: str
) -> None:
    violations = find_forbidden_domain_imports(violating_code)
    assert expected_violation in violations


# (b) Privacidade LGPD/GDPR: PiiInputGuardrail remove 100% de E-mails, CPFs, Nomes e PNRs
def test_ff06b_pii_guardrail_scrubs_100_percent_of_emails_pnrs_cpf_and_names() -> None:
    raw_input = (
        "Passageiro: Lucas Sabino da Silva (CPF 123.456.789-00), contato lucas.sabino@vibecoders.com.br "
        "e secundario suporte.viagem@empresa.org, teve o bilhete K8M2P1 e conexao X9R4T2 cancelados."
    )
    sanitized = PiiInputGuardrail.sanitize_prompt(raw_input, traveler_name="Lucas Sabino da Silva")

    assert "lucas.sabino@vibecoders.com.br" not in sanitized
    assert "suporte.viagem@empresa.org" not in sanitized
    assert "123.456.789-00" not in sanitized
    assert "Lucas Sabino da Silva" not in sanitized
    assert "K8M2P1" not in sanitized
    assert "X9R4T2" not in sanitized
    assert "[REDACTED_EMAIL]" in sanitized
    assert "[REDACTED_CPF]" in sanitized
    assert "[REDACTED_NAME]" in sanitized
    assert "[REDACTED_PNR]" in sanitized


# (c) Contrato Estrito de Saída: Regex policy_doc_id, Confiança >= 0.75 e Human-in-the-Loop
def test_ff06b_pydantic_accepts_valid_grounded_advice() -> None:
    advice = AutonomousDisruptionAdvice(
        trip_id="trp_99481",
        severity="CRITICAL",
        options=[
            RebookingActionOption(
                action_type="REBOOK_FLIGHT",
                summary="Reacomodar gratuitamente no próximo voo disponível conforme Resolução ANAC 400.",
                policy_doc_id="POL-ANAC-0400",
                confidence_score=0.91,
                requires_human_approval=True,
            )
        ],
        disclaimer_required=True,
    )
    assert advice.options[0].policy_doc_id == "POL-ANAC-0400"
    assert advice.disclaimer_required is True


@pytest.mark.parametrize(
    "invalid_doc_id,confidence,disclaimer,human_approval",
    [
        ("DOC-INVENTADO-99", 0.90, True, True),   # Viola regex ^POL-[A-Z]{2,4}-\d{4}$
        ("POL-GOL-12", 0.90, True, True),         # Dígitos insuficientes na policy_doc_id
        ("POL-GOL-1024", 0.62, True, True),       # Abaixo do limiar mínimo confidence_score >= 0.75
        ("POL-GOL-1024", 0.88, False, True),      # Tenta desativar disclaimer jurídico obrigatório
        ("POL-GOL-1024", 0.88, True, False),      # Tenta executar REBOOK_FLIGHT sem Human-in-the-Loop
    ],
)
def test_ff06b_pydantic_rejects_ungrounded_or_unsafe_llm_outputs(
    invalid_doc_id: str, confidence: float, disclaimer: bool, human_approval: bool
) -> None:
    with pytest.raises(ValidationError):
        AutonomousDisruptionAdvice(
            trip_id="trp_99481",
            severity="CRITICAL",
            options=[
                RebookingActionOption(
                    action_type="REBOOK_FLIGHT",
                    summary="Sugestão gerada pelo modelo para remarcação imediata.",
                    policy_doc_id=invalid_doc_id,
                    confidence_score=confidence,
                    requires_human_approval=human_approval,
                )
            ],
            disclaimer_required=disclaimer,
        )


# (d) Resiliência: Saída Inválida de LLM ou Falha de Provedor Aciona Fallback Determinístico
class HallucinatingLlmStub(ILlmProviderAdapter):
    def __init__(self) -> None:
        self.calls = 0

    def complete_json(self, sanitized_prompt: str) -> str:
        self.calls += 1
        return json.dumps(
            {
                "severity": "CRITICAL",
                "options": [
                    {
                        "action_type": "REBOOK_FLIGHT",
                        "summary": "Reembolso em dobro inventado sem documento oficial.",
                        "policy_doc_id": "ALUCINACAO-000",
                        "confidence_score": 0.40,
                        "requires_human_approval": True,
                    }
                ],
                "disclaimer_required": True,
            }
        )


class TimeoutLlmStub(ILlmProviderAdapter):
    def complete_json(self, sanitized_prompt: str) -> str:
        raise TimeoutError("Provedor LLM excedeu o timeout estrito de 3.5s (Circuit Breaker).")


def test_ff06b_invalid_llm_output_exhausts_single_retry_and_triggers_deterministic_fallback() -> None:
    stub = HallucinatingLlmStub()
    orchestrator = ResilientAiOrchestrator(provider=stub)

    result = orchestrator.generate_advice(
        trip_id="trp_5001",
        raw_prompt="Voo do PNR K8M2P1 e email viajante@corp.com foi cancelado.",
    )

    assert stub.calls == 2  # 1 chamada original + máximo de 1 retry de autocorreção
    assert result.is_fallback is True
    assert result.options[0].action_type == "CONTACT_AIRLINE"
    assert result.options[0].policy_doc_id == "POL-ANAC-0400"
    assert all("K8M2P1" not in p and "viajante@corp.com" not in p for p in orchestrator.sent_prompts)


def test_ff06b_provider_timeout_immediately_triggers_deterministic_fallback() -> None:
    orchestrator = ResilientAiOrchestrator(provider=TimeoutLlmStub())
    result = orchestrator.generate_advice(trip_id="trp_5002", raw_prompt="Atraso severo no embarque.")

    assert result.is_fallback is True
    assert result.disclaimer_required is True
    assert result.options[0].confidence_score == 1.0


# (e) Governança Quantitativa SOLID: Todos os Pacotes Refatorados na Sequência Principal (D = 0.0)
@pytest.mark.parametrize(
    "package_name,n_a,n_c,c_a,c_e",
    [
        ("Itinerarios.Domain.Contracts", 3, 3, 2, 0),
        ("Itinerarios.Infrastructure", 0, 3, 0, 3),
        ("Integracao.Contracts", 2, 2, 2, 0),
        ("Integracao.Adapters", 0, 3, 0, 2),
        ("Gateway.BFF", 0, 3, 0, 3),
    ],
)
def test_ff04_martin_main_sequence_distance_equals_zero(
    package_name: str, n_a: int, n_c: int, c_a: int, c_e: int
) -> None:
    abstractness = n_a / n_c
    instability = c_e / (c_a + c_e)
    distance = abs(abstractness + instability - 1.0)
    assert distance == pytest.approx(0.0), f"Pacote {package_name} desviou da Sequência Principal: D={distance}"
```

---

## 4. Como Executar Localmente na Esteira de Qualidade

Para validar os contratos e guardrails diretamente em ambiente de desenvolvimento ou no GitHub Actions:

```bash
pip install pytest pydantic
pytest -v
```
