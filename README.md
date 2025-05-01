**Premissas:**

1.  **Foco Educacional:** A plataforma *não* gerencia pacientes reais, nem administra cannabis. Ela serve como repositório de conhecimento, material didático e visualização de dados *anonimizados* provenientes das fases de pesquisa (fora da escola).
2.  **Segurança e Privacidade:** Dados clínicos são estritamente anonimizados antes de serem expostos na plataforma educacional. O acesso a dados mais sensíveis (mesmo anonimizados) pode ser restrito a perfis específicos (pesquisadores, professores).
3.  **Modularidade:** A arquitetura permite adicionar novos módulos e funcionalidades no futuro.

---

## I. Backend (FastAPI)

**Tecnologias:**

*   **Framework:** FastAPI
*   **Linguagem:** Python 3.9+
*   **Banco de Dados:** PostgreSQL (robusto para dados relacionais e JSONB)
*   **ORM:** SQLAlchemy (com `asyncpg` para suporte assíncrono)
*   **Migrações:** Alembic
*   **Validação de Dados:** Pydantic (integrado ao FastAPI)
*   **Autenticação:** JWT (JSON Web Tokens) para proteger endpoints
*   **Containerização (Opcional, recomendado):** Docker, Docker Compose

**Estrutura de Diretórios (Exemplo):**

```
cannabis-edu-backend/
├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   ├── endpoints/
│   │   │   │   ├── auth.py
│   │   │   │   ├── users.py
│   │   │   │   ├── chemovars.py
│   │   │   │   ├── rare_diseases.py
│   │   │   │   ├── clinical_data.py  # Dados anonimizados
│   │   │   │   ├── educational_content.py
│   │   │   │   └── feedback.py
│   │   │   └── api.py             # Agregador de rotas v1
│   │   └── deps.py                # Dependências (ex: usuário atual)
│   ├── core/
│   │   ├── config.py              # Configurações (env vars)
│   │   └── security.py            # Funções de hash e JWT
│   ├── crud/                      # Operações CRUD no DB
│   │   ├── crud_user.py
│   │   ├── crud_chemovar.py
│   │   └── ... (outros cruds)
│   ├── db/
│   │   ├── base.py                # Base declarativa do SQLAlchemy
│   │   ├── session.py             # Configuração da sessão do DB
│   │   └── init_db.py             # (Opcional) Script para popular DB inicial
│   ├── models/                    # Modelos SQLAlchemy (tabelas)
│   │   ├── user.py
│   │   ├── chemovar.py
│   │   ├── rare_disease.py
│   │   ├── anonymized_clinical_data.py
│   │   ├── educational_module.py
│   │   ├── feedback.py
│   │   └── ...
│   ├── schemas/                   # Modelos Pydantic (validação API)
│   │   ├── user.py
│   │   ├── chemovar.py
│   │   ├── token.py
│   │   └── ...
│   └── main.py                    # Ponto de entrada da aplicação FastAPI
├── alembic/                       # Configurações e versões de migração
├── tests/                         # Testes unitários e de integração
├── .env                           # Variáveis de ambiente (não versionar)
├── .gitignore
├── alembic.ini
├── docker-compose.yml             # (Opcional)
├── Dockerfile                     # (Opcional)
└── requirements.txt
```

**Modelos de Dados Principais (Simplificado):**

1.  **`User` (models/user.py, schemas/user.py):**
    *   `id`, `email`, `hashed_password`, `full_name`, `role` (e.g., 'student', 'teacher', 'researcher', 'admin'), `is_active`, `school_id` (opcional, FK para `School`).
2.  **`Chemovar` (models/chemovar.py, schemas/chemovar.py):**
    *   `id`, `name` (ou código), `description`, `cbd_percentage`, `thc_percentage`, `other_cannabinoids` (JSONB?), `terpene_profile` (JSONB?), `potential_therapeutic_uses`.
3.  **`RareDisease` (models/rare_disease.py, schemas/rare_disease.py):**
    *   `id`, `name`, `description`, `cid_code` (opcional), `symptoms`, `current_treatments_info`.
4.  **`AnonymizedClinicalData` (models/anonymized_clinical_data.py, schemas/clinical_data.py):**
    *   `id`, `study_id` (referência interna), `rare_disease_id` (FK), `chemovar_id` (FK), `data_point_description` (e.g., "Redução média de crises epilépticas"), `metric_name` (e.g., "percent_reduction"), `metric_value` (e.g., 45.5), `dose_info` (e.g., "10mg CBD/dia"), `observed_effects` (texto descritivo anonimizado), `data_source` (e.g., "Ensaio Clínico X - Hospital Y").
    *   **Importante:** *Nenhum* dado que possa identificar um paciente. Foco em resultados agregados ou métricas anonimizadas.
5.  **`EducationalModule` (models/educational_module.py, schemas/educational_content.py):**
    *   `id`, `title`, `subject` (e.g., 'Biologia', 'Química', 'Ética'), `target_grade` (e.g., 'Ensino Médio'), `content` (Markdown, HTML ou JSON), `associated_media_urls` (JSONB?), `related_chemovars` (M2M?), `related_diseases` (M2M?).
6.  **`Activity` (models/activity.py, schemas/educational_content.py):**
    *   `id`, `module_id` (FK), `type` (e.g., 'simulation', 'quiz', 'debate_prompt', 'interview_guide'), `description`, `instructions`, `resource_url` (opcional).
7.  **`Feedback` (models/feedback.py, schemas/feedback.py):**
    *   `id`, `user_id` (FK, opcional se anônimo), `module_id` (FK, opcional), `activity_id` (FK, opcional), `rating` (int), `comments` (text), `submission_timestamp`.

**Endpoints da API (Exemplos - `app/api/v1/endpoints/`):**

*   **Autenticação (`auth.py`):**
    *   `POST /api/v1/login`: Autentica usuário, retorna token JWT.
    *   `POST /api/v1/register`: (Opcional) Registro de novos usuários (pode precisar de aprovação).
*   **Chemovars (`chemovars.py`):**
    *   `GET /api/v1/chemovars`: Lista chemovars (filtrável por CBD/THC, etc.).
    *   `GET /api/v1/chemovars/{chemovar_id}`: Detalhes de um chemovar.
    *   `POST /api/v1/chemovars`: Cria novo chemovar (requer role 'admin' ou 'researcher').
*   **Doenças Raras (`rare_diseases.py`):**
    *   `GET /api/v1/rare-diseases`: Lista doenças raras.
    *   `GET /api/v1/rare-diseases/{disease_id}`: Detalhes de uma doença.
*   **Dados Clínicos Anonimizados (`clinical_data.py`):**
    *   `GET /api/v1/clinical-data`: Lista dados anonimizados (filtrável por doença, chemovar). *Acesso pode ser restrito por role*.
    *   `GET /api/v1/clinical-data/summary`: Retorna dados agregados para visualização (e.g., média de eficácia por doença/chemovar).
*   **Conteúdo Educacional (`educational_content.py`):**
    *   `GET /api/v1/modules`: Lista módulos (filtrável por matéria, série).
    *   `GET /api/v1/modules/{module_id}`: Detalhes de um módulo (inclui atividades).
    *   `GET /api/v1/activities/{activity_id}`: Detalhes de uma atividade.
    *   `POST /api/v1/modules`: Cria novo módulo (requer role 'admin' ou 'teacher').
*   **Feedback (`feedback.py`):**
    *   `POST /api/v1/feedback`: Submete feedback sobre um módulo/atividade.
    *   `GET /api/v1/feedback`: Lista feedback (requer role 'admin' ou 'teacher').

**Considerações Adicionais:**

*   **CORS:** Configurar `CORSMiddleware` para permitir requisições do frontend React.
*   **Testes:** Essenciais para garantir a funcionalidade e segurança.
*   **Logging:** Implementar logging robusto para monitoramento e depuração.

---

## II. Frontend (React)

**Tecnologias:**

*   **Framework/Biblioteca:** React 17+
*   **Linguagem:** JavaScript (ES6+) ou TypeScript (recomendado para projetos maiores)
*   **Gerenciador de Pacotes:** npm ou yarn
*   **Bundler:** Vite ou Create React App (Vite é geralmente mais rápido)
*   **Roteamento:** React Router DOM (`react-router-dom`)
*   **Requisições HTTP:** Axios ou Fetch API
*   **Gerenciamento de Estado (Opcional, recomendado):** Redux Toolkit, Zustand ou Context API (para estados globais como usuário logado, tema)
*   **UI Kit:** Material UI (MUI), Ant Design, Chakra UI (escolha um para consistência visual)
*   **Visualização de Dados:** Recharts, Chart.js (com wrapper React), Nivo
*   **Estilização:** CSS Modules, Styled Components, Tailwind CSS (integrado com o UI Kit escolhido)

**Estrutura de Diretórios (Exemplo):**

```
cannabis-edu-frontend/
├── public/
│   └── index.html
├── src/
│   ├── assets/              # Imagens, fontes, etc.
│   ├── components/          # Componentes reutilizáveis (Button, Card, Modal)
│   │   ├── common/
│   │   ├── layout/          # Header, Footer, Sidebar
│   │   └── specific/        # ChemovarCard, ModuleListItem, DataChart
│   ├── contexts/            # (Opcional) Context API providers (AuthContext)
│   ├── features/            # (Alternativa) Organizar por funcionalidade
│   │   ├── auth/
│   │   ├── modules/
│   │   └── ...
│   ├── hooks/               # Custom Hooks (useAuth, useApi)
│   ├── pages/               # Componentes de página (rotas principais)
│   │   ├── LoginPage.jsx
│   │   ├── RegisterPage.jsx
│   │   ├── DashboardPage.jsx
│   │   ├── ModulesListPage.jsx
│   │   ├── ModuleDetailPage.jsx
│   │   ├── ChemovarsPage.jsx
│   │   ├── RareDiseasesPage.jsx
│   │   ├── ClinicalDataVizPage.jsx # Página de visualização
│   │   ├── CommunityPage.jsx       # Workshops, etc.
│   │   └── NotFoundPage.jsx
│   ├── services/            # Lógica de chamada da API (api.js, authService.js)
│   ├── store/               # (Opcional) Configuração do Redux/Zustand
│   ├── styles/              # Estilos globais, temas
│   ├── types/               # (Se usar TypeScript) Definições de tipos
│   ├── utils/               # Funções utilitárias
│   ├── App.jsx              # Componente raiz com roteamento
│   └── main.jsx             # Ponto de entrada da aplicação React
├── .env                     # Variáveis de ambiente (ex: URL da API)
├── .gitignore
├── index.html               # (Se usar Vite)
├── package.json
├── vite.config.js           # (Se usar Vite)
└── README.md
```

**Componentes e Funcionalidades Chave:**

1.  **Autenticação:**
    *   Páginas de Login/Registro (`LoginPage`, `RegisterPage`).
    *   Serviço de autenticação (`authService.js`) para chamar a API de login/registro.
    *   Gerenciamento do token JWT (armazenar em `localStorage` ou `sessionStorage`).
    *   Contexto de Autenticação (`AuthContext`) ou estado global para armazenar informações do usuário logado e status de autenticação.
    *   Rotas Protegidas: Componente que verifica se o usuário está logado antes de renderizar páginas restritas.
2.  **Layout:**
    *   Componentes `Header`, `Sidebar` (se aplicável), `Footer`.
    *   Navegação principal baseada no `role` do usuário.
3.  **Conteúdo Educacional (`ModulesListPage`, `ModuleDetailPage`):**
    *   Listar módulos com filtros (matéria, série).
    *   Exibir conteúdo do módulo (texto, imagens, vídeos).
    *   Listar e linkar para atividades associadas (simulações, quizzes - podem ser links externos ou componentes interativos).
    *   Componente para submeter feedback (`FeedbackForm`).
4.  **Informações Científicas (`ChemovarsPage`, `RareDiseasesPage`):**
    *   Listar e exibir detalhes sobre chemovars (perfis químicos, usos potenciais).
    *   Listar e exibir detalhes sobre doenças raras (descrição, sintomas).
    *   Possibilidade de relacionar visualmente doenças e chemovars relevantes.
5.  **Visualização de Dados Clínicos (`ClinicalDataVizPage`):**
    *   **Crucial:** Usar bibliotecas de gráficos (Recharts, Chart.js) para exibir os dados *anonimizados* da API (`/api/v1/clinical-data/summary`).
    *   Exemplos: Gráficos de barras comparando eficácia média de diferentes chemovars para uma doença; gráficos de linha mostrando curvas dose-resposta (abstratas, baseadas em dados agregados); tabelas resumindo efeitos colaterais observados (frequências anonimizadas).
    *   Filtros interativos para selecionar doença, chemovar, métrica.
    *   **Ênfase:** Deixar claro que são dados de pesquisa anonimizados para fins educacionais.
6.  **Engajamento Comunitário (`CommunityPage`):**
    *   Seção com informações sobre workshops para pais/professores.
    *   Links para recursos externos, ONGs parceiras.
    *   Fórum simples (se desejado, aumenta complexidade).
7.  **Gerenciamento (Admin):**
    *   (Se necessário) Páginas separadas ou seções em páginas existentes (protegidas por role) para criar/editar módulos, chemovars, usuários.

**Considerações Adicionais:**

*   **Responsividade:** Garantir que a interface funcione bem em diferentes tamanhos de tela (desktop, tablet, mobile).
*   **Acessibilidade (a11y):** Seguir as diretrizes WCAG (semântica HTML, contraste de cores, navegação por teclado).
*   **Performance:** Otimizar o carregamento (code splitting, lazy loading de componentes/imagens).
*   **Estado Global:** Escolher a estratégia de gerenciamento de estado adequada à complexidade.
*   **Tratamento de Erros:** Exibir mensagens de erro amigáveis para o usuário quando chamadas de API falham.

---

**Fluxo de Interação Básico:**

1.  Usuário (aluno, professor) acessa o frontend.
2.  É redirecionado para a página de Login.
3.  Insere credenciais. O frontend envia para `POST /api/v1/login` no backend.
4.  Backend valida, gera JWT e retorna ao frontend.
5.  Frontend armazena o JWT e informações do usuário (no estado global/contexto).
6.  Usuário navega para a página de Módulos.
7.  Frontend faz uma requisição `GET /api/v1/modules` (enviando o JWT no header `Authorization`).
8.  Backend verifica o JWT, busca os módulos no DB e retorna a lista.
9.  Frontend exibe a lista de módulos.
10. Usuário clica em um módulo. Frontend busca detalhes `GET /api/v1/modules/{id}`.
11. Usuário acessa a página de Visualização de Dados. Frontend busca dados `GET /api/v1/clinical-data/summary`. Backend retorna dados agregados e anonimizados. Frontend renderiza os gráficos.

---

Este detalhamento fornece uma base sólida para iniciar o desenvolvimento. Cada fase do projeto original se reflete em diferentes partes da aplicação: a Fase 1 e 2 alimentam os dados (Chemovars, ClinicalData), a Fase 3 é implementada principalmente no Frontend (Módulos, Atividades, Visualização) e a Fase 4 pode ser suportada pela coleta de Feedback e análise de uso (se implementado).
