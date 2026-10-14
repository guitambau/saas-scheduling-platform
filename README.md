# Scheduling SaaS — Plataforma de Agendamento Multi-Tenant

> Projeto prático do Software Engineering Lab para estudar arquitetura de software, isolamento multi-tenant, modelagem de domínio, concorrência e qualidade de código.

## Visão geral

O **Scheduling SaaS** é uma plataforma de agendamento pensada para que diferentes empresas gerenciem seus serviços, profissionais e horários em um único produto, mantendo seus dados isolados.

Este é um **projeto de estudo em desenvolvimento**. As funcionalidades e decisões descritas a seguir representam o escopo planejado, e não uma implementação já concluída.

## Objetivos técnicos

- Projetar um **monólito modular** com fronteiras claras entre contextos de negócio.
- Implementar **multi-tenancy** com isolamento consistente em todas as operações.
- Modelar regras de agendamento, disponibilidade, cancelamento e prevenção de conflitos.
- Explorar transações, concorrência, constraints e índices no PostgreSQL.
- Aplicar autenticação, autorização e segurança desde a camada de domínio até a persistência.
- Construir testes automatizados e uma estratégia reproduzível de execução e deploy.
- Instrumentar a aplicação com logs estruturados, métricas e rastreamento distribuído quando pertinente.

## Escopo funcional — MVP

| Módulo | Responsabilidades |
| --- | --- |
| Empresas (tenants) | Cadastro, configuração, status e fuso horário da empresa. |
| Usuários e acesso | Autenticação, vínculo com empresa e perfis de acesso. |
| Serviços | Cadastro, duração e disponibilidade de serviços. |
| Profissionais | Cadastro e associação aos serviços oferecidos. |
| Agenda | Horários de atendimento, bloqueios e consulta de disponibilidade. |
| Agendamentos | Criar, consultar, reagendar e cancelar com validação de conflitos. |

**Fora do MVP:** pagamentos, notificações por WhatsApp/e-mail, integrações com calendário e marketplace público. Esses itens podem entrar em etapas futuras.

## Arquitetura proposta

**Abordagem inicial:** monólito modular, com uma aplicação Spring Boot e separação por módulos de negócio. O objetivo é exercitar limites bem definidos sem introduzir complexidade operacional desnecessária.

**Modelo inicial de multi-tenancy:** PostgreSQL compartilhado, schema compartilhado e `tenant_id` nas entidades pertencentes a uma empresa.

```text
Cliente / API Consumer
        |
        v
Autenticação e autorização
        |
        v
Resolução do tenant autenticado
        |
        v
API Spring Boot (módulos de negócio)
        |
        v
Serviços de aplicação e domínio
        |
        v
Persistência com escopo do tenant
        |
        v
PostgreSQL
```

### Regras essenciais do multi-tenant

1. O tenant utilizado nas operações protegidas será derivado da identidade autenticada e do vínculo autorizado do usuário; **não** será confiado a um `tenant_id` arbitrário enviado pelo cliente.
2. Leituras, atualizações e exclusões de registros pertencentes a empresas devem incluir o escopo do tenant.
3. Relacionamentos entre registros do domínio devem impedir associações entre empresas diferentes, com validação na aplicação e, sempre que viável, garantias no banco.
4. Índices e restrições deverão considerar `tenant_id` quando a regra de negócio for específica da empresa.
5. Testes de isolamento entre dois tenants serão obrigatórios, incluindo tentativas de acessar IDs de outra empresa.
6. Operações administrativas globais, se existirem, terão permissões e fluxos explicitamente separados.

## Regras de agendamento

- Cada agendamento pertence a uma empresa, serviço e profissional.
- O horário solicitado deve respeitar a duração do serviço, a jornada e os bloqueios do profissional.
- Um profissional não poderá ter agendamentos ativos sobrepostos.
- Alterações e cancelamentos seguirão regras de negócio explícitas, com histórico quando necessário.
- Datas serão armazenadas como instantes consistentes, considerando o **fuso horário configurado para o tenant** na apresentação e no cálculo de disponibilidade.
- A prevenção de agendamentos duplicados deve resistir a **requisições simultâneas**, usando transações e garantias no banco, não apenas consultas prévias.

## Stack proposta

| Camada | Tecnologia |
| --- | --- |
| Linguagem | Java 25 |
| Framework | Spring Boot |
| Persistência | PostgreSQL, Spring Data JPA |
| Versionamento do banco | Flyway |
| Segurança | Spring Security |
| Testes | JUnit, Testcontainers |
| Ambiente local | Docker Compose |
| Documentação da API | OpenAPI |
| Observabilidade | Micrometer / OpenTelemetry (fase posterior) |

As tecnologias podem mudar conforme os resultados dos experimentos.

## Estrutura planejada

```text
saas-scheduling-platform/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── .../
│   │   │       ├── tenant/
│   │   │       ├── identity/
│   │   │       ├── catalog/
│   │   │       ├── staff/
│   │   │       ├── availability/
│   │   │       └── booking/
│   │   └── resources/
│   │       └── db/migration/
│   └── test/
├── docs/
│   └── adr/
├── compose.yaml
└── README.md
```

*A estrutura é ilustrativa; será ajustada conforme o projeto evoluir.*

## Plano de implementação

- [ ] **Fase 1 — Fundação:** estrutura modular, banco, migrations, configuração e testes iniciais.
- [ ] **Fase 2 — Multi-tenancy:** empresas, vínculo de usuários, resolução e isolamento de tenant.
- [ ] **Fase 3 — Catálogo:** serviços, profissionais, jornadas e indisponibilidades.
- [ ] **Fase 4 — Agendamentos:** criação, consulta, reagendamento, cancelamento e validação de conflitos.
- [ ] **Fase 5 — Robustez:** concorrência, índices, testes de integração e cenários de segurança.
- [ ] **Fase 6 — Operação:** observabilidade, CI/CD, documentação e preparação de deploy.

## Critérios de qualidade

O projeto será considerado tecnicamente consistente quando:

- O isolamento entre tenants for coberto por testes automatizados.
- Dois agendamentos conflitantes não puderem ser confirmados em condições de concorrência.
- As invariantes do domínio forem aplicadas em um local claro e verificável.
- As migrations permitirem reconstruir o banco de forma reproduzível.
- As decisões arquiteturais importantes estiverem documentadas em ADRs.
- A aplicação puder ser executada e testada localmente de maneira reproduzível.

## Execução local

As instruções de execução serão adicionadas após a implementação da primeira fase, quando existirem `compose.yaml`, migrations e comandos Maven validados. Até lá, este documento descreve o **projeto planejado**.

## Aprendizados e decisões

O objetivo não é apenas entregar uma API funcional, mas registrar **trade-offs**: por que usar monólito modular, como garantir isolamento, onde impor restrições, como lidar com concorrência e o que medir antes de otimizar.

As decisões relevantes serão documentadas em `docs/adr/` conforme forem tomadas.

---

**Software Engineering Lab** — construir, testar, medir e documentar.